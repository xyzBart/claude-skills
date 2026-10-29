---
name: install-game-sniper-ghost-warrior
description: Use when installing or troubleshooting the GOG version of Sniper: Ghost Warrior (2010, City Interactive) under Wine on this machine. Covers the full confirmed install procedure and fixes for serial-key popups, tiny installer UI, DXVK adapter failure, and the black-screen-with-sound bug. Check this before re-diagnosing from scratch.
version: 1.0.0
allowed-tools: Bash, Read, Edit, WebSearch
---

# Sniper: Ghost Warrior (GOG) on Wine

Game lives at `~/mediaCenter/games/SniperGhostWarrior/` with its own dedicated win32 Wine
prefix at `~/mediaCenter/games/SniperGhostWarrior/.wine` (see [[project_gog_wine_gaming]] memory
for the general GOG/Wine library layout on this machine). GOG installer files are the
`setup_sniper_ghost_warrior_*.exe` + `.bin` parts, originally downloaded to `~/Downloads`.

Confirmed working end-to-end on: Ubuntu 24.04, Wine 9.0, Intel Iris Xe (Mesa 25.2.8), 2026-10-29.

## Install procedure (confirmed working order)

1. **Do not use `innoextract` alone.** It only unpacks files and skips the installer's own
   script logic — which for this game includes silently writing a valid serial into the
   registry (a holdover from its old CD-key protection; GOG doesn't expose this serial anywhere
   in your account, it's meant to be invisible). Extracting with `innoextract` only and then
   launching the game produces a **"serial key invalid"** popup with no key available anywhere
   to enter. Fix: always run the real installer through Wine instead:
   ```
   export WINEPREFIX=~/mediaCenter/games/SniperGhostWarrior/.wine
   export WINEARCH=win32
   wineboot --init
   wine "setup_sniper_ghost_warrior_1.2_(34044).exe"
   ```
   (The `.bin` sibling files must stay next to the `.exe` — the installer finds them
   automatically.) Click through the GUI install wizard yourself; point the install path at
   `C:\GOG Games\Sniper Ghost Warrior` (default is fine, it lands inside the prefix which is
   already under `~/mediaCenter/games/SniperGhostWarrior/`).

2. **If the installer UI renders tiny** (HiDPI screen, Wine defaults to 96 DPI): bump the
   prefix DPI before launching the installer:
   ```
   wine reg add "HKEY_CURRENT_USER\Control Panel\Desktop" /v LogPixels /t REG_DWORD /d 144 /f
   ```

3. **Install the 32-bit GStreamer codec plugins and native Windows Media components** — the
   game's menu plays a WMV/ASF background video that neither Wine's bundled GStreamer ASF
   support nor its stub `wmvcore.dll` handle correctly out of the box:
   ```
   sudo apt-get install -y gstreamer1.0-plugins-bad:i386 gstreamer1.0-libav:i386 gstreamer1.0-plugins-ugly:i386
   winetricks -q wmp9
   ```
   `wmp9` overrides Wine's `wmvcore`/`quartz`/`wmp` DLLs with the real Microsoft ones. This
   clears the `wmvcore:wm_reader_read_stream_sample` / `GStreamer-Video-CRITICAL` errors you'll
   otherwise see in the launch log. Confirmed it does **not** by itself fix the black screen
   (see below) — it's a separate, real bug, but worth doing anyway for a clean log and working
   intro video.

4. **Install DXVK, but pin it to 1.10.3 — do not use the latest.** DXVK 2.0+ requires Vulkan
   1.3 and will print `DXVK: No adapters found. Please check your device filter settings` on
   this machine's Intel driver (missing the `khrLoadStoreOpNone` feature it demands), silently
   falling back to a broken state. 1.10.3 is the last version before that requirement and fully
   supports D3D9:
   ```
   winetricks -q dxvk1103
   ```
   Without DXVK, the game also runs (wined3d/OpenGL) but is prone to the same black-screen
   symptom below plus worse performance — DXVK is still worth having installed.

## The black-screen-with-sound bug (the big one)

**Symptom**: game launches, you hear intro/menu audio and can navigate (click sounds), but the
video output is solid black. This is **not** a Wine/DXVK/driver bug — it reproduces on native
Windows too and is a well-documented bug in this specific 2010 engine build. Don't waste time
on wined3d-vs-DXVK swapping or GPU driver theories; confirmed via two independent proofs:
- Enabling `DXVK_HUD=full` shows real draw calls / non-zero FPS / GPU utilization happening —
  so the renderer is working and producing frames.
- The game's own in-engine screenshot function (bound to a hotkey, saves to
  `~/Documents/Sniper - Ghost Warrior/out/ScreenShots/*.tga`) captures a genuinely solid-black
  frame — so it's the engine's own render content that's black, not a Wine/X11 compositing
  issue.

**Fix**:
```
rm -f ~/Documents/"Sniper - Ghost Warrior"/out/Settings/Video.scr
rm -f ~/Documents/"Sniper - Ghost Warrior"/out/Settings/Audio.scr
```
Then relaunch **without** wrapping it in a Wine virtual desktop (`wine explorer /desktop=...`)
— the game's own config expects real fullscreen and launching it windowed/virtual-desktop-wrapped
was a red herring tried during diagnosis, not part of the actual fix. Just:
```
export WINEPREFIX=~/mediaCenter/games/SniperGhostWarrior/.wine
export WINEARCH=win32
cd ~/mediaCenter/games/SniperGhostWarrior/.wine/drive_c/"GOG Games"/"Sniper Ghost Warrior"
wine Sniper_x86.exe
```
This regenerates fresh `Video.scr`/`Audio.scr` on first run and the game renders correctly.

## Resolution doesn't fill the monitor

The regenerated `Video.scr` defaults to `Resolution(1368,768)` (or similar) regardless of your
actual display. Edit it to match:
```
~/Documents/Sniper - Ghost Warrior/out/Settings/Video.scr
```
Change the `Resolution(w,h)` line to your monitor's native resolution (check with
`xrandr | grep '\*'`), keep `Fullscreen()` as-is, then relaunch. On this machine the monitor is
2560x1440.

## Quick launch (after everything above is done once)

```
export WINEPREFIX=~/mediaCenter/games/SniperGhostWarrior/.wine
export WINEARCH=win32
cd ~/mediaCenter/games/SniperGhostWarrior/.wine/drive_c/"GOG Games"/"Sniper Ghost Warrior"
wine Sniper_x86.exe
```
