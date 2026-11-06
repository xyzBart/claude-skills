---
name: fix-desktop-resume-lag
description: Use when the laptop feels laggy for a while after resuming from suspend on uzer-Latitude-5420 (Dell Latitude 5420, Ubuntu 24.04, s2idle-only suspend). Covers the confirmed root causes — a burst of Persistent=yes systemd timers firing on resume, and anacron scheduling the full cron.daily/cron.weekly batch on day-crossing resumes — and the fixes applied for each.
version: 1.0.0
allowed-tools: Bash, Read
---

# Laptop lag after resume (Dell Latitude 5420, Ubuntu 24.04)

Machine: uzer-Latitude-5420, s2idle-only suspend (`cat /sys/power/mem_sleep` →
`[s2idle]`). The symptom is general sluggishness for a short period right
after waking from suspend — not a single app, the whole desktop.

## Symptom

Desktop feels laggy for a bit after resuming from suspend/sleep.

## Root causes found (two, confirmed across two investigations)

### 1. `Persistent=yes` systemd timers piling up on resume (2026-09-13)

On a laptop that suspends/resumes many times a day, timers with
`Persistent=yes` frequently miss their scheduled slot while asleep, so
systemd fires them **immediately on resume** instead of waiting for the next
slot. Several firing in the same second as WiFi reconnect + compositor
redraw is what shows up as post-resume lag.

Confirmed live: at a 16:54:18 resume, `fwupd-refresh`, `sysstat-collect`, and
an `anacron` attempt all fired within the same second as suspend-exit.

Timers with `Persistent=yes` on this box: `apt-daily`, `apt-daily-upgrade`,
`man-db`, `fstrim`, `fwupd-refresh`, `anacron`, `e2scrub_all`,
`dpkg-db-backup`, `logrotate`.

Specific problems identified:
- `fwupd-refresh.service` failed on **every single resume** (100%
  reproducible) — races the network before WiFi reassociates, always errors
  out. Wasted work + journal spam every resume.
- `vm.swappiness=60` (Ubuntu default) with 31GB RAM and an 8GB swap file —
  more proactive swap-out while idle means more swap-*in* stalls when
  resuming and touching those apps again.
- `e2scrub_all` and `man-db` occasionally do several seconds of real
  disk/CPU work and aren't time-critical to run same-day.

#### Diagnose

```bash
journalctl --since "<resume timestamp>" --until "+90 seconds"
systemctl list-timers --all | grep -i persistent   # cross-check with: systemctl show <timer> -p Persistent
```

#### Fix applied

```bash
# 1. Disable fwupd-refresh.timer (check firmware manually instead, on AC + connected)
sudo systemctl disable --now fwupd-refresh.timer
sudo fwupdmgr refresh   # run manually when wanted

# 2. Lower swappiness
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf

# 3. Drop Persistent= on e2scrub_all.timer and man-db.timer
sudo mkdir -p /etc/systemd/system/e2scrub_all.timer.d /etc/systemd/system/man-db.timer.d
printf '[Timer]\nPersistent=false\n' | sudo tee /etc/systemd/system/e2scrub_all.timer.d/override.conf
printf '[Timer]\nPersistent=false\n' | sudo tee /etc/systemd/system/man-db.timer.d/override.conf
sudo systemctl daemon-reload
```

Left untouched (`Persistent=yes` kept, judged low-impact — each completes in
a few seconds and provides real value): `apt-daily`, `apt-daily-upgrade`,
`fstrim`, `anacron` (itself; see below for its *triggered batch*),
`dpkg-db-backup`, `logrotate`.

### 2. `anacron` firing the full `cron.daily`/`cron.weekly` batch on day-crossing resumes (2026-10-02)

The fixes above held up for same-day resumes, but lag persisted specifically
on resumes that crossed midnight. `anacron.timer` runs hourly and just
checks "has a calendar day passed since cron.daily last ran?". On a
same-day resume it finds nothing to do and exits instantly. On a resume that
crosses midnight, it detects the new day and — if on AC power — schedules
the **entire `cron.daily` batch** (`man-db` regen, `logrotate`, `dpkg`
cleanup, `apt-compat`, `sysstat` summary, `apport`, `lighttpd` maint) to run
**5 minutes later, sequentially, as one job**, right while the user is back
at the keyboard, at normal CPU/IO priority (`Nice=0`, best-effort IO) —
directly competing with the foreground session.

Confirmed in logs: the batch was gated behind `ConditionACPower=true` across
~9 hours of on-battery resumes, then fired 5 minutes after plugging in
triggered the condition, running `cron.daily` then `cron.weekly` back to
back.

#### Diagnose

```bash
journalctl -u anacron.service --since "<resume date> 00:00:00" --no-pager
```

#### Fix applied

Low-priority override so the batch no longer competes with the interactive
session for CPU/disk when it does run:

```bash
sudo mkdir -p /etc/systemd/system/anacron.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/anacron.service.d/override.conf
[Service]
Nice=19
IOSchedulingClass=idle
CPUWeight=10
IOWeight=10
EOF
sudo systemctl daemon-reload
```

## Noise ruled out as non-issues (seen during investigation, left alone)

- thermald `Unable to find a zone for STG1` — benign Dell ACPI naming
  quirk, no perf impact.
- gnome-shell `Cursor update failed: drmModeAtomicCommit: Invalid argument`
  on resume — known cosmetic i915/mutter bug.
- Bluetooth resuming cleanly — not implicated here; if BT itself fails to
  come back after resume, see the separate `bluetooth-resume-fix` skill
  instead.
- `snap.firefox.firefox`'s `MemoryPoller` denied by apparmor reading
  `/proc/pressure/memory` every 5 seconds, continuously, 24/7. Harmless
  per-event, wasted syscalls/journal writes, not resume-specific.
- One-off kernel `WARNING: typec_altmode_update_active` from the USB-C
  dock/hub (Realtek RTS5411) renegotiating on resume, ~2x/day. Known
  harmless upstream kernel bug.

## Still-open suspect if lag recurs after both fixes

A GNOME session that's been running continuously for weeks (53 days when
last observed) is a plausible independent source of gradual sluggishness
(JS heap growth, shader cache bloat) that resume just makes noticeable.
Not fixed by anything above — if lag still appears on a day-change resume
after both fixes, test with a logout/login or reboot before assuming a new
root cause.

## Usage history

- **2026-09-13**: diagnosed and fixed the `Persistent=yes` timer pile-up
  (fwupd-refresh disabled, swappiness lowered, e2scrub_all/man-db
  Persistent dropped).
- **2026-10-02**: lag still reported on day-change resumes after the above;
  found and fixed the `anacron`-triggered `cron.daily` batch competing at
  normal priority (added `Nice=19`/`IOSchedulingClass=idle` override).
