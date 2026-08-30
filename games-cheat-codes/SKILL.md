---
name: games-cheat-codes
description: Use when the user asks for cheat codes/console commands for a Steam game on uzer-Latitude-5420, or wants to combine cheat launch options with an existing Proton fix (see steam-proton-fixes). Growing per-game reference of confirmed console-enable steps and cheat commands.
version: 1.0.0
allowed-tools: Bash, Read, WebSearch, WebFetch
---

# Game cheat codes (uzer-Latitude-5420)

Reference of confirmed cheat-code/console setups per game. Check the case list below before
searching fresh.

## Combining with Steam Launch Options — ordering gotcha

Steam Launch Options are a shell command line where `%command%` is substituted with the actual
game launch command.

- Anything **before** `%command%` is shell prefix: environment variables (`PROTON_OLD_GL_STRING=1`,
  `PROTON_LOG=1`, `WINEDEBUG=...`) or wrapper commands.
- Anything meant as an **argument to the game itself** (engine cvars like `+set sv_cheats 1`)
  must go **after** `%command%`. Put them before it and the shell tries to literally execute
  `+set` as a program, and the game fails to start at all (not a hang, not a crash — it never
  launches).

Correct pattern:
```
<ENV_VARS>=1 %command% +set cvar1 value1 +set cvar2 value2
```

See [[steam-proton-fixes]] for the matching Proton/launch-options troubleshooting skill — combine
a game's Proton fix env vars with its cheat cvars in one line using this ordering.

## Case history

### Call of Duty: United Offensive (appid 2640) — CONFIRMED 2026-08-30

Engine: id Tech 3 derivative (same console/cvar system as base Call of Duty).

**Enable console + cheats** — Launch Options:
```
+set thereisacow 1337 +set developer 1 +set sv_cheats 1 +set monkeytoy 0
```
Combined with this game's Proton fix (PROTON_OLD_GL_STRING=1, see steam-proton-fixes case
history), the full working Launch Options line is:
```
PROTON_OLD_GL_STRING=1 %command% +set thereisacow 1337 +set developer 1 +set sv_cheats 1 +set monkeytoy 0
```
Then press `~` in-game to open the console.

**Cheat commands**:
| Command | Effect |
|---|---|
| `god` | Invincibility |
| `give all` | All weapons |
| `give health` | Full health refill |
| `notarget` | Enemies ignore you |
| `noclip` | Walk through walls |
| `kill` | Suicide/reset |
| `map_restart` | Restart current level |
| `devmap <mapname>` | Load a specific map in dev/cheat mode (e.g. `devmap bastogne1`) |

**Note**: cheats only work on a resumed/saved game. On a fresh game, save right after loading,
then quit and reload before using console commands.

Sources: GameFAQs, Call of Duty Fandom wiki (Developer console), Neoseeker — see
steam-proton-fixes case entry conversation for links.

<!-- Add new games below, same format: Enable console / Cheat commands / Notes -->
