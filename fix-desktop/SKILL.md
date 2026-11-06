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

### New launchers: "Notes Any Quick" / "Notes IT" — open notes vaults in VSCodium, raising the right window from either icon or terminal

**Status**: permanent, added 2026-10-22.
**Files**: `~/.local/share/applications/notes-any-quick.desktop`,
`~/.local/share/applications/notes-it.desktop`.
**Final working config**: each is a `Type=Application` launcher —
`Exec=codium --ozone-platform=x11 <vault-path>` (`/home/uzer/notes/any` and
`/home/uzer/notes/IT`), `Icon=vscodium`, `StartupNotify=false`, and a
per-launcher dummy `StartupWMClass` (`notes-any-quick-codium-launcher` /
`notes-it-codium-launcher`) that never matches a real window — see problem 2
below for why. Both share the normal default VSCodium profile/singleton; no
`--user-data-dir` needed. Ran `update-desktop-database
~/.local/share/applications/` after any edit.

Two stacked problems, found by testing (terminal-launched `codium <path>`
always worked correctly, even mid-investigation — that's what pointed at
GNOME Shell's app-matching rather than Codium itself):

1. **Not coming to front at all** (desktop icon *and*, at first, terminal) —
   Codium is Electron and defaults to running as a *native Wayland* client,
   which can't raise/focus its own window without a proper xdg-activation
   token. Same root cause as the Firefox `DISABLE_WAYLAND=1` fix above;
   `--ozone-platform=x11` is the Electron/Chromium equivalent, forcing
   XWayland where the older X11 raise/activate protocol works reliably with
   Mutter. Also applied to the user-override
   `~/.local/share/applications/codium.desktop` (shadows the system one,
   same flag on both the main entry and the `new-empty-window` action) and
   to `alias codium='command codium --ozone-platform=x11'` in `~/.bashrc`,
   so every entry point (icons, "Open With", terminal) is consistent.
   **Caveat**: only takes effect on a fresh process start — a VSCodium
   already running natively on Wayland needs a full quit (close all
   windows) once before this applies.

2. **Icon-click only: clicking one vault's icon raised the OTHER vault's
   window instead of opening/focusing its own** — confirmed via `xprop`
   that Codium's main editor window's `WM_CLASS` is hardcoded to
   `VSCodium` regardless of any `--class=` flag passed (only some internal
   hidden window picks up `--class`). First guess was a PID/process
   tracking problem, "fixed" by giving each vault its own
   `--user-data-dir` (separate singleton-lock scope → genuinely separate
   process per vault) — **that alone did not fix it**, proving the window
   tracker isn't PID-based for this decision. The real mechanism: GNOME
   Shell's "is this app already open?" check runs *before* `Exec` is even
   invoked, and matches purely on the declared `StartupWMClass` string
   against currently-visible windows. Both launchers declared
   `StartupWMClass=VSCodium` — identical to the real window class and to
   each other — so clicking either one while *any* Codium window was open
   short-circuited straight to focusing that existing window; `Exec` for
   the second launcher never even ran. (This is also why terminal always
   worked: a bare `codium <path>` skips this Shell pre-check entirely and
   goes straight to Codium's own — correct — folder-aware window routing.)
   **Fix**: set each launcher's `StartupWMClass` to a value that will
   *never* match a real Codium window, so Shell's pre-check never finds a
   match and always actually runs `Exec`, handing off to Codium's own
   correct single-instance/folder-aware routing. Once this was in place,
   `--user-data-dir` turned out to be unnecessary and was removed — the
   dummy `StartupWMClass` was the actual fix, not the separate profile.
   **Cosmetic tradeoff**: because the declared class never matches the
   real window, these windows may show up ungrouped/generic-icon in
   alt-tab or any app-grouped view — functionally harmless, just not
   grouped under a named app there.

### digiKam: UI text too small, doesn't scale with GNOME's "Large Text" accessibility setting

**Status**: permanent, added 2026-11-06.
**File**: `~/.local/share/applications/digikam.desktop`, `Exec=` line.
**Fix**: `Exec=bash -c 'QT_FONT_DPI=140 /home/uzer/progs/digikam'`

digiKam here runs as a Qt6 AppImage (`~/progs/digiKam-9.0.0-Qt6-x86-64.appimage`,
symlinked from `~/progs/digikam`). This machine's GNOME "Large Text" toggle
sets `org.gnome.desktop.interface text-scaling-factor` to `1.25`, which
gnome-settings-daemon propagates to X11/XWayland apps as `Xft.dpi: 120`
(`96 * 1.25`) in the X resource database — confirmed via `xrdb -query`. Qt's
`xcb` platform backend (digiKam falls back to `xcb`; the log shows `Could
not find the Qt platform plugin "wayland"`) reads `Xft.dpi` automatically,
so digiKam already renders at 120 DPI *without* any override — not 96 as
first assumed. That's why an initial fix attempt of `QT_FONT_DPI=120` was a
no-op (identical to the automatic baseline, so no visible change) and caused
a long back-and-forth misdiagnosing "the fix isn't working." Confirmed the
env var does work at all by testing an exaggerated `QT_FONT_DPI=240`
(visibly huge, incl. splash screen) — proved the mechanism, not the value,
needed adjusting. Bisected by hand with a terminal loop (`for v in 96 110
125 140 150 165; do QT_FONT_DPI=$v /home/uzer/progs/digikam; done`, closing
each window to advance) to find a size the user actually liked: **140**,
above the automatic 120 but short of full doubling.
**To re-tune if "Large Text" or the monitor setup changes**: rerun that
bisection loop and edit the `QT_FONT_DPI=` number in the `Exec=` line.
**Caveat**: only takes effect on a fresh process start, same as the Codium
fix above — fully quit any running digiKam first.

<!-- Add new small fixes below, same format: Status / File / Fix / why -->
