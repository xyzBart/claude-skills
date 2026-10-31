---
name: install-game
description: Use when installing or troubleshooting any Windows game (GOG, standalone installer, etc.) under Wine on this machine — general technique that applies before reaching for a game-specific skill. Covers the D3D-version check that should happen before a game's first launch, which DXVK/d8vk version to use, and general crash/black-screen diagnostic approach. Check for a dedicated per-game skill (e.g. install-game-<name>) first if one exists; this is the fallback/general knowledge.
version: 1.0.0
allowed-tools: Bash, Read, Edit, WebSearch
---

# Installing Windows games under Wine (uzer-Latitude-5420)

Machine: Dell Latitude 5420, Ubuntu 24.04.4 LTS, kernel 6.17, 11th Gen Intel Core i5-1145G7,
Intel Iris Xe Graphics (TigerLake-LP GT2) with Mesa/i915 driver. No discrete GPU. Same machine
covered by [[steam-proton-fixes]] (for Steam/Proton titles specifically) — this skill is for
non-Steam Windows games run directly through plain Wine (GOG installers, standalone .exe, etc.).

Games live under `~/mediaCenter/games/<GameName>/`, each with its own dedicated Wine prefix at
`~/mediaCenter/games/<GameName>/.wine`. See [[project_gog_wine_gaming]] memory for the library
layout and running list of gotchas; per-game skills (`install-game-<name>`) hold full confirmed
procedures for specific titles already solved — check for one of those before diagnosing from
scratch. Existing ones: [[install-game-sniper-ghost-warrior]], [[install-game-sniper-elite-berlin-1945]].

## Before the first launch: check the D3D version, pick the right translation layer

**Do this before running any game for the first time**, especially anything older than ~2012.
Running an old D3D game on Wine's default `wined3d` (native OpenGL translation) risks a
genuine GPU hang on this machine's Intel driver — not just a game crash, but one that can freeze
the *entire desktop* (GNOME Shell's compositor shares the same GPU context and gets stuck on the
same wedged fence) and force a hard power-off. Confirmed with Sniper Elite - Berlin 1945
(D3D8): `i915 ... GPU HANG ... in wined3d_cs`, failed driver reset, then
`Fence expiration time out ... gnome-shell[...]`.

1. Check which Direct3D version the game uses:
   ```
   strings <GameExe>.exe | grep -E '^d3d[0-9]+\.dll$|^ddraw\.dll$'
   ```
2. Install the matching Vulkan translation layer **before** the first real launch:
   - **D3D9 / D3D10 / D3D11** → DXVK. Install via `winetricks -q dxvk1103` — **pin to 1.10.3,
     do not use the `dxvk` latest verb.** DXVK 2.0+ requires Vulkan 1.3 and fails outright on
     this machine's driver (`DXVK: No adapters found` — missing the `khrLoadStoreOpNone`
     feature). 1.10.3 is the last version before that requirement.
   - **D3D8** → `d8vk`. Not in winetricks' catalog, and mainline DXVK's later-merged D3D8
     support requires the same Vulkan 1.3 that fails here. Download the standalone
     `AlpyneDreams/d8vk` v1.0 release manually:
     ```
     gh release download d8vk-v1.0 --repo AlpyneDreams/d8vk --pattern "d8vk-v1.0.tar.gz"
     tar xzf d8vk-v1.0.tar.gz -C <dir>
     cp <dir>/x32/d3d8.dll <prefix>/drive_c/windows/system32/d3d8.dll
     wine reg add "HKEY_CURRENT_USER\Software\Wine\DllOverrides" /v d3d8 /d native /f
     ```
   - **D3D7/DirectDraw or earlier** → no known-good Vulkan layer confirmed on this machine yet;
     treat as higher-risk for the same GPU-hang class of bug, test cautiously.
3. After installing, confirm the right renderer is active on first launch: set
   `DXVK_LOG_LEVEL=info DXVK_LOG_PATH=<some file>` and check for a `DXVK: v...` or
   `DXVK: d8vk-v...` banner in the log. If it's missing, the DLL override didn't take.

## Older GOG installers: don't use innoextract alone

For titles with legacy CD-key/serial protection (common pre-~2015 re-releases), unpacking the
`.exe` with `innoextract` alone skips the installer's embedded script logic — including a step
that silently writes a pre-filled valid serial into the registry. Symptom: "serial key invalid"
popup with no key available anywhere (GOG doesn't expose it; it's meant to be invisible).
**Always run the actual installer through Wine** (`wine setup_*.exe`, click through the GUI
yourself) for older titles instead.

## Black screen (with working audio) troubleshooting order

Don't jump straight to deep Wine/DXVK driver debugging. Check in this order:
1. The game's own cached video/display settings file (often `My Documents/<game>/.../Settings`
   or similar) — try deleting it so it regenerates with fresh defaults. This class of bug is
   often a documented issue even on native Windows, not Wine-specific.
2. Enable `DXVK_HUD=full` — if it shows real draw calls / non-zero FPS / GPU utilization, the
   renderer is working and the bug is in the game/engine, not Wine's graphics stack.
3. If the game has its own in-engine screenshot function, use it and inspect the saved file
   directly — compare against what's visible on screen to rule out a Wine/X11 compositing issue
   versus genuinely black render output.

## DPI / small installer UI

Wine prefixes default to 96 DPI, which renders old installers' UI tiny on this laptop's HiDPI
screen. Bump it just for the install if needed:
```
wine reg add "HKEY_CURRENT_USER\Control Panel\Desktop" /v LogPixels /t REG_DWORD /d 144 /f
```
Consider resetting it back to default (delete the key) after install for the game itself — it
can interfere with some old games' own resolution auto-detection.

## Desktop launchers

GOG/Wine installers usually auto-create a menu entry under
`~/.local/share/applications/wine/Programs/<Game> [GOG.com]/` — check there before creating a
new one (avoids duplicate entries in GNOME's app search). For a Desktop icon, copy/adapt that
`.desktop` file, point `Exec` at the game's exe directly (more reliable than a `.lnk` shortcut),
and mark it trusted so it launches without a GNOME warning:
```
gio set ~/Desktop/"<Game>.desktop" metadata::trusted true
```
