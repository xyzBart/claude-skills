---
name: install-game-sniper-elite-berlin-1945
description: Use when installing or troubleshooting the GOG version of Sniper Elite - Berlin 1945 (the original 2005 Sniper Elite, re-released under this title) under Wine on this machine. Covers the install procedure, a critical fix for a full-system-crashing GPU hang (d8vk), and a known cosmetic aspect-ratio quirk. Check this before re-diagnosing from scratch.
version: 1.0.0
allowed-tools: Bash, Read, Edit, WebSearch
---

# Sniper Elite - Berlin 1945 (GOG) on Wine

See [[install-game]] for general technique (D3D-version check before first launch, DXVK/d8vk
pinning, hardware/OS baseline) that applies across all Wine game installs on this machine.

Game lives at `~/mediaCenter/games/SniperEliteBerlin1945/` with its own dedicated win32 Wine
prefix at `~/mediaCenter/games/SniperEliteBerlin1945/.wine` (see [[project_gog_wine_gaming]]
memory for the general GOG/Wine library layout on this machine). This is the original 2005
**Sniper Elite** (Rebellion Developments), re-released on GOG under the title "Berlin 1945"
since it's set during the Battle of Berlin — not a separate sequel.

Confirmed working end-to-end on: Ubuntu 24.04, Wine 9.0, Intel Iris Xe (Mesa 25.2.8), 2026-10-30.

## Install procedure

Run the real GOG installer through Wine (same reasoning as
[[install-game-sniper-ghost-warrior]] — avoid `innoextract`-only unpacking for older titles):
```
export WINEPREFIX=~/mediaCenter/games/SniperEliteBerlin1945/.wine
export WINEARCH=win32
wineboot --init
wine reg add "HKEY_CURRENT_USER\Control Panel\Desktop" /v LogPixels /t REG_DWORD /d 144 /f  # optional, only for a more readable installer UI
wine "setup_sniper_elite_-_berlin_1945_1.0_(19500).exe"
```
This installer did not hit the serial-key gotcha that Ghost Warrior did — ran clean end to end.
Installs to `C:\GOG Games\Sniper Elite - Berlin 1945`, main exe is `SniperElite.exe`.

## CRITICAL: install d8vk before ever running this game — do not skip

**This game uses Direct3D 8** (confirmed via `strings SniperElite.exe | grep d3d8`), which Wine
renders by default through `wined3d` — its native-OpenGL D3D translation. Running it that way
on this machine triggered an **unrecoverable Intel i915 GPU hang that froze the entire desktop
and required a hard power-off to recover**:
```
kernel: i915 0000:00:02.0: [drm] GPU HANG: ecode 12:1:84dffffb, in wined3d_cs
kernel: i915 0000:00:02.0: [drm] *ERROR* GT0: rcs0 reset request timed out
kernel: Fence expiration time out i915-...:gnome-shell[...]
```
The driver's own hang-recovery failed, and because GNOME Shell's Mutter compositor shares the
same GPU context, it got stuck waiting on a fence that never completed too — taking down the
whole desktop, not just the game. Confirmed via `journalctl -b -1 -k | grep -i i915` after the
crash.

**Fix — install `d8vk`** (Direct3D 8→Vulkan translation, same idea as DXVK but for D3D8, which
stock DXVK does not cover):
```
gh release download d8vk-v1.0 --repo AlpyneDreams/d8vk --pattern "d8vk-v1.0.tar.gz"
tar xzf d8vk-v1.0.tar.gz -C <extract-dir>
cp <extract-dir>/x32/d3d8.dll ~/mediaCenter/games/SniperEliteBerlin1945/.wine/drive_c/windows/system32/d3d8.dll
export WINEPREFIX=~/mediaCenter/games/SniperEliteBerlin1945/.wine
wine reg add "HKEY_CURRENT_USER\Software\Wine\DllOverrides" /v d3d8 /d native /f
```
Not in winetricks' catalog — has to be pulled directly from GitHub releases. Not part of
mainline DXVK either (as of the 1.10.3 era used for Ghost Warrior); d8vk later got merged into
DXVK upstream but only in versions requiring Vulkan 1.3, which fails outright on this machine's
driver (see [[install-game-sniper-ghost-warrior]] for that same Vulkan-1.3 adapter-detection
failure) — so use the standalone `AlpyneDreams/d8vk` v1.0 release, not mainline DXVK's D3D8
support.

After installing d8vk, confirmed dozens of minutes of play with zero GPU errors in
`journalctl -k -b 0`.

## Launch

No virtual-desktop wrapper needed — runs fine natively fullscreen:
```
export WINEPREFIX=~/mediaCenter/games/SniperEliteBerlin1945/.wine
export WINEARCH=win32
cd ~/mediaCenter/games/SniperEliteBerlin1945/.wine/drive_c/"GOG Games"/"Sniper Elite - Berlin 1945"
wine SniperElite.exe
```

## Known quirks (not fixable, don't waste time chasing them further)

- **No in-game Windowed/Fullscreen toggle** — only a resolution dropdown. Max listed resolution
  is 1600x1200 (4:3) or lower options under a separate 16:9 aspect setting (e.g. 864x486) —
  genuinely hardcoded, confirmed no registry/config override exists (settings save to a binary
  file `Prof/<hash>/prof.prf`, not safely hand-editable).
- **Scope/HUD ring renders as an oval, not a circle, in BOTH 4:3 and 16:9 modes** — stretched in
  *opposite* directions (16:9: wider-than-tall; 4:3: taller-than-wide), roughly equal magnitude.
  Confirmed via pixel measurement on screenshots, and confirmed the 4:3 case persists even when
  the Wine window size exactly matches the game's requested backbuffer size (no presenter
  scaling mismatch involved) — so this is **not** a Linux/Wine/d8vk scaling bug, it's the
  original engine's own 2D HUD rendering at some fixed internal reference aspect that doesn't
  exactly match either 4:3 or 16:9. Cosmetic only — crosshair center and all 3D geometry are
  accurate regardless. Don't re-chase this; it was tested across multiple resolutions and virtual
  desktop configs with consistent results.
- To fill the whole screen without distortion, pick whichever aspect mode the game offers that
  matches your monitor's native ratio (16:9 on this machine) and run natively fullscreen (no
  Wine virtual desktop) — the GPU-side scaling will then be uniform.
