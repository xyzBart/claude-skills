---
name: steam-proton-fixes
description: Use when a Steam/Proton game on uzer-Latitude-5420 won't launch, freezes, crashes, or shows graphical corruption. Covers general Linux/Proton diagnostic technique for this machine (process-tree/livelock checks, Wine SEH logging, launch-option gotchas) plus a growing per-game case history of confirmed fixes. Check the case history first for a game that's already been solved before re-diagnosing from scratch.
version: 1.0.0
allowed-tools: Bash, Read, WebSearch, WebFetch
---

# Steam/Proton game troubleshooting (uzer-Latitude-5420)

Machine: Dell Latitude 5420, Ubuntu 24, GNOME Shell/Wayland, Mesa Intel Iris Xe (11th Gen
i5-1145G7). Steam install lives at `~/.steam/debian-installation` (not the default
`~/.steam/steam`).

## Check the case history first

Before doing a fresh diagnosis, check the "Case history" section below — the specific
won't-launch/freeze/crash issue for a given game may already be documented with a confirmed fix.

## General diagnostic technique (learned from the Call of Duty: United Offensive case)

- **Launch-options gotcha**: editing Steam's `userdata/<id>/config/localconfig.vdf` directly
  while the Steam client is running does **not** work — the running client keeps its own
  in-memory copy and silently ignores/overwrites on-disk edits. Always set Launch Options
  through Steam's own Properties → General → Launch Options dialog instead.
- **To get a real Wine debug log**: set Launch Options to
  `PROTON_LOG=1 WINEDEBUG=+seh,+tid %command%`, reproduce, then read `$HOME/steam-<appid>.log`.
  This file can get huge (100s of MB–1GB+) if the game livelocks — delete it once you've
  extracted what you need, don't leave it sitting in the user's home directory.
- **Don't invoke `steam.sh steam://rungameid/<id>` directly from a shell** to "launch it
  yourself" — it spawns a brand-new nested Steam sandbox instance instead of talking to the
  already-running client, and its own bwrap/user-namespace check can fail outright ("Steam now
  requires user namespaces to be enabled"), producing an unrelated error dialog. Ask the user to
  click Play in the already-open client instead.
- **A frozen/hung game process is not necessarily waiting.** Check `ps -o pid,pcpu,stat` for the
  actual game exe. Sustained high %CPU with state `R` means it's spinning (livelock, e.g. a SEH
  exception-handling loop), not blocked. Confirm blocked-vs-spinning before chasing a "network
  hang" or "deadlock" theory:
  - `ss -tnp` / `ss -xp` for the process tree — look for `SYN_SENT` (real network stall) vs.
    nothing (not network-related).
  - `/proc/<pid>/wchan` and `/proc/<pid>/status` (`State: R` vs `S` vs `D` vs `Z`) to see if it's
    actually blocked on something or actively running.
- **Screenshots**: GNOME's Screenshot D-Bus portal (`org.gnome.Shell.Screenshot`) and X11
  root-window capture via `import`/XWayland are **not** available non-interactively in this
  environment (`AccessDenied` / `Resource temporarily unavailable`). Ask the user to take the
  screenshot themselves (saved to `~/Pictures/Screenshots/`) and read the resulting file instead
  of trying to capture the screen directly.
- **Compat tools already installed here**: Proton Experimental, Proton 7.0, Proton 8.0, Proton
  Hotfix, GE-Proton (install by downloading the official GitHub release + verifying its
  sha512sum, extracting into `~/.steam/debian-installation/compatibilitytools.d/`, then
  restarting Steam so it picks up the new tool).
- **ProtonDB and the ValveSoftware/Proton GitHub issue tracker** are good first stops for a
  specific game+appid — search "<game name> Proton" and check
  `github.com/ValveSoftware/Proton/issues` for an issue titled with the game's name/appid before
  deep-diagnosing from scratch. Fetch full issue threads with
  `gh api repos/ValveSoftware/Proton/issues/<n>/comments` — WebFetch alone only returns the issue
  body, not comments, and ProtonDB's page is a JS SPA that WebFetch can't render, so rely on
  search-result snippets rather than fetching protondb.com directly.

## Case history

### Call of Duty: United Offensive (appid 2640) — SOLVED 2026-08-21

**Symptom**: goes fullscreen (black screen), hangs completely, GNOME eventually shows
"steam_app_2640 is not responding — Force Quit/Wait". COD (2003, appid 2620) and COD 2 launch
fine from the same shared install folder.

**Fix**: Launch Options → `PROTON_OLD_GL_STRING=1 %command%`

**Root cause**: the engine's early hardware-detection code copies the GPU's OpenGL extension
string into a fixed-size buffer sized for 2004-era GPUs. Modern drivers return a much longer
string, overflowing the buffer (a write access violation, `code=c0000005`), and Wine's exception
dispatch can't recover from the corrupted state — instead of crashing cleanly it livelocks (CPU
pinned ~96%, spinning between a corrupted jump target and a handler stub, forever). That's what
shows up as "frozen black screen."

**What was ruled out along the way** (don't re-try these for this game):
- Network/CD-key validation hang — no `SYN_SENT` sockets, no network activity tied to the
  process during the freeze.
- `com_recommendedSet` cvar / first-run autoconfig — tested `+set com_recommendedSet 0`,
  identical crash.
- Proton version — Proton 7.0, 8.0, and GE-Proton11-5 all hit the byte-identical fault
  address/signature. Not a Proton regression.
  - GE-Proton did improve the outer symptom even before the real fix was found: clean ~30s
    self-teardown instead of an indefinite hang requiring Force Quit.
- CPU topology — `WINE_CPU_TOPOLOGY=1:0` (masking down to 1 logical CPU), identical crash.
- Found via two Valve/Proton GitHub issues for this exact game/engine family (#3776 "Call of
  Duty: United Offensive (2640)", #2507 for the base game) documenting `PROTON_OLD_GL_STRING=1`
  fixing an identical "won't launch"/"buffer overrun" symptom class.

<!-- Add new games below, same format: Symptom / Fix / Root cause (if known) / What was ruled out -->
