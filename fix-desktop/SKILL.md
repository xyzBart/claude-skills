---
name: fix-desktop
description: Use when applying, documenting, or looking up a small manual fix/tweak to this machine's desktop environment (Ubuntu 24.04 LTS "noble", GNOME Shell 46, Wayland session) on uzer-Latitude-5420 — e.g. desktop launcher Exec tweaks, GNOME settings quirks, small app-level workarounds. This is the log of small fixes so they can be reapplied after a failure or Ubuntu reinstall. For bigger, self-contained problem areas there are dedicated skills (e.g. steam-proton-fixes, bluetooth-resume-fix) and legacy Claude Projects (dell-latitude-5420, hp-probook-460-g11) — check those first/instead for anything in their scope; don't duplicate here.
version: 1.0.0
allowed-tools: Bash, Read, Edit
---

# Desktop fixes (Ubuntu 24.04 LTS / GNOME Shell 46, uzer-Latitude-5420)

Running log of small, manual fixes/tweaks applied to this specific desktop
install so they can be **reapplied after a failure or a fresh Ubuntu
reinstall**. Machine: Dell Latitude 5420, Ubuntu 24.04.4 LTS ("noble"), GNOME
Shell 46.0, Wayland session.

Scope: *small* fixes — a one-off `Exec=` edit, a GNOME setting, a dconf
tweak, a small config file change. If a problem turns out to be big enough to
need its own diagnostic writeup (multiple symptoms, case history, non-obvious
root cause), it gets its own skill instead of living here — see below.

## Don't duplicate — check these first

- **steam-proton-fixes** skill (`~/.claude/skills/steam-proton-fixes/`) —
  Steam/Proton game launch/crash/freeze issues.
- **fix-window-positioning** skill (`~/.claude/skills/fix-window-positioning/`) —
  app opening on the wrong monitor (laptop panel vs BenQ external); confirmed
  root cause + fix pattern, demonstrated on IntelliJ IDEA.
- **bluetooth-resume-fix** skill (`~/.claude/skills/bluetooth-resume-fix/`) —
  Bluetooth not coming back after suspend/resume.
- **~/repos/env/desktop/dell-latitude-5420/** (legacy Claude Project, not yet
  a skill) — external monitor/BenQ blackout issues, lid-close action, suspend
  /resume, printer (CUPS/avahi), Mozilla file-drop. Has its own `monitor/` and
  `printer/` subfolders with dated incident writeups.
- **~/repos/env/desktop/hp-probook-460-g11/** (legacy Claude Project, not yet
  a skill) — printer config dump/fix for that machine.

These will likely get merged into one consolidated desktop-fixes skill
eventually (this one, steam-proton-fixes, bluetooth-resume-fix, and the two
Claude Projects above) — until then, treat them as separate, check all of
them when diagnosing, and don't copy their content in here.

## Fixes

### Firefox: force XWayland (`DISABLE_WAYLAND=1`) — fixes context menu showing in wrong position

**Status**: permanent, already applied (predates this log).
**File**: `~/.local/share/applications/firefox_firefox.desktop`, `Exec=` line.
**Fix**: `Exec=env DISABLE_WAYLAND=1 WAYLAND_DISPLAY= <firefox-binary> %u`

This session runs native Wayland (`XDG_SESSION_TYPE=wayland`). GTK's native
Wayland popup positioning (`xdg-popup`, relative-to-parent-surface only) has
had long-standing bugs placing context menus/popups at the wrong spot.
Forcing Firefox onto XWayland (empty `WAYLAND_DISPLAY` + `DISABLE_WAYLAND=1`)
sidesteps it since X11 allows absolute window positioning. Reapply this env
pair to whatever binary path is in the `Exec=` line if the desktop file is
ever regenerated (e.g. by a snap refresh) or recreated from scratch.

### Firefox: launcher pointed at `~/progs/firefox` build instead of the snap

**Status**: TEMPORARY (as of 2026-10-18) — revert once done testing the
`~/progs` build.
**File**: `~/.local/share/applications/firefox_firefox.desktop`.
**What changed**: the main `[Desktop Entry]` `Exec=` line was repointed from
`/snap/bin/firefox %u` to `/home/uzer/progs/firefox/firefox %u` (the
`DISABLE_WAYLAND=1`/`WAYLAND_DISPLAY=` prefix above was kept). The original
snap `Exec=` line is left commented out directly above the new one for easy
revert. The three `Desktop Action` entries (new window / new private window
/ profile manager) were **not** changed and still launch the snap build.
**To revert**: uncomment the `/snap/bin/firefox %u` line, comment out (or
delete) the `~/progs` line.

<!-- Add new small fixes below, same format: Status / File / Fix / why -->
