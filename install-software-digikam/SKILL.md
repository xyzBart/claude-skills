---
name: install-software-digikam
description: Use when installing, launching, or troubleshooting digiKam on uzer-Latitude-5420 — it runs as a Qt6 AppImage, not a package install. Covers the current install layout, the font-scaling fix for GNOME's "Large Text" setting, and a startup-speed fix for the AppImage's FUSE/squashfs overhead. Check this before re-diagnosing a digiKam launcher or startup issue from scratch.
version: 1.0.0
allowed-tools: Bash, Read, Edit
---

# digiKam (Qt6 AppImage) on uzer-Latitude-5420

digiKam is **not** installed via apt/snap/flatpak here — it runs as a
downloaded Qt6 AppImage, launched through a desktop file.

## Current layout

- Binary: `~/progs/digiKam-9.0.0-Qt6-x86-64.appimage` (307MB, the original
  downloaded AppImage — kept as-is, still works standalone).
- Convenience symlink: `~/progs/digikam` → the `.appimage` file above (for
  ad-hoc terminal use; safe to invoke through this symlink, see gotcha
  below for why that's *not* true of the extracted `AppRun`).
- Extracted copy (for startup speed, see below):
  `~/progs/digiKam-9.0.0-Qt6-x86-64.appimage.extracted/` (852MB
  uncompressed), created with `./digiKam-9.0.0-Qt6-x86-64.appimage
  --appimage-extract` then `mv squashfs-root
  digiKam-9.0.0-Qt6-x86-64.appimage.extracted`.
- Launcher: `~/.local/share/applications/digikam.desktop`:
  ```
  Exec=bash -c 'QT_FONT_DPI=140 /home/uzer/progs/digiKam-9.0.0-Qt6-x86-64.appimage.extracted/AppRun'
  ```
  (run `update-desktop-database ~/.local/share/applications/` after editing)

Confirmed working end-to-end on: Ubuntu 24.04, GNOME Shell 46, Wayland
session, digiKam 9.0.0 Qt6 AppImage, 2026-11-06.

## Fix 1: UI text too small, doesn't scale with GNOME's "Large Text" setting

**Symptom**: GNOME Settings → Accessibility → "Large Text" is on
(`org.gnome.desktop.interface text-scaling-factor` = `1.25`), other apps
scale up, digiKam's text looks unaffected.

**Root cause**: this machine's GNOME setting propagates to X11/XWayland
apps as `Xft.dpi: 120` (`96 * 1.25`) in the X resource database (confirmed
via `xrdb -query`). Qt's `xcb` platform backend — digiKam falls back to
`xcb`, the startup log shows `Could not find the Qt platform plugin
"wayland"` — reads `Xft.dpi` automatically, so digiKam already renders at
120 DPI **without any override**, not at a bare 96 as first assumed. That
made an initial fix attempt of `QT_FONT_DPI=120` a no-op (identical to the
already-active automatic baseline, so no visible change) — a long
back-and-forth misdiagnosing "the fix isn't working" before this was
caught. Confirmed the `QT_FONT_DPI` env var does work at all by testing an
exaggerated `QT_FONT_DPI=240` (visibly huge, including the splash screen)
— proved the mechanism was fine and only the chosen value needed fixing.

**Fix**: `QT_FONT_DPI=140` in the `Exec=` line (see current `Exec=` above).
Bisected by hand with a terminal loop, closing each window to advance to
the next value:
```bash
for v in 96 110 125 140 150 165; do
  QT_FONT_DPI=$v /home/uzer/progs/digikam
done
```
140 — above the automatic 120 baseline but short of full doubling — is what
the user picked as the sweet spot.

**To re-tune** if "Large Text" is toggled off, or the scaling factor
changes: rerun that bisection loop and edit the `QT_FONT_DPI=` number in
`Exec=`.

**Caveat**: only takes effect on a fresh process start — fully quit any
running digiKam first.

## Fix 2: ~10s slow startup (FUSE/squashfs AppImage overhead)

**Symptom**: ~6s to show the splash screen, ~10s total before the main
window is usable.

**Root cause**: launching the `.appimage` file directly runs it through a
FUSE-mounted squashfs (`/tmp/.mount_digikaXXXXXX` — AppImage's standard
runtime mode). Every library/plugin load during startup — the full bundled
Qt6 + KDE Frameworks stack (platform plugins, imageformats, icon engines,
styles), then the bundled FFmpeg libs for Qt Multimedia — pays a FUSE
round-trip + squashfs decompression cost. Confirmed by capturing
per-line timestamped startup logs:
```bash
START=$(date +%s.%N)
timeout 20 bash -c 'QT_FONT_DPI=140 /home/uzer/progs/digikam' 2>&1 \
  | while IFS= read -r line; do printf "%s %s\n" "$(date +%s.%N)" "$line"; done \
  > digikam_timed.log
```
This showed two ~3.8s dead-air gaps, right where the dynamic linker loads
those library sets — not explained by disk speed (`~/progs` is on NVMe,
confirmed via `lsblk ROTA=0`). Full timeline and numbers in
`~/kasuj/digikamStart.md`.

**Fix**: extract the AppImage once to a real directory and launch the
extracted `AppRun` instead of the `.appimage` file — skips FUSE/squashfs
entirely on every subsequent launch:
```bash
cd ~/progs
./digiKam-9.0.0-Qt6-x86-64.appimage --appimage-extract
mv squashfs-root digiKam-9.0.0-Qt6-x86-64.appimage.extracted
```
Then point `Exec=` at `.../digiKam-9.0.0-Qt6-x86-64.appimage.extracted/AppRun`.

**Result**: preload→platform-init gap dropped from +3.76s to +0.07s; the
gap before the FFmpeg/splash-gone line dropped from +3.8s to +1.87s; total
time to last init line roughly halved, ~10.5s → ~5.0s.

**Gotcha — never invoke the extracted `AppRun` through a symlink**: it's a
plain bash script that locates its own AppDir with `dirname "$0"` (line 92
of `AppRun`) — not symlink-aware. Pointing `Exec=` at
`~/progs/digikam` (the convenience symlink, which targets the original
`.appimage`) made `AppRun` think its AppDir was `~/progs` itself, failing
with `digikam: command not found`. The original `.appimage` file doesn't
have this problem since the AppImage runtime is a compiled binary that
resolves its own location more robustly. **Always call the extracted
`AppRun`'s real path directly in `Exec=`**; leave the `~/progs/digikam`
symlink pointed at the `.appimage` file for ad-hoc terminal use, where it's
unaffected.

**Trade-off**: ~852MB extra disk space for the extracted copy (uncompressed;
negligible here, 340GB free).

**To re-apply after a digiKam AppImage version upgrade**: delete the old
`*.appimage.extracted/` dir, re-run the extract+rename above with the new
version number, and update the `Exec=` line (both fixes' DPI value and the
binary path) to match.
