---
name: fix-window-positioning
description: Use when an app keeps opening on the wrong monitor on uzer-Latitude-5420 (dual-monitor: laptop panel + BenQ external, Ubuntu 24.04, GNOME Shell 46, Wayland). Covers the confirmed root cause (per-app saved window state defaulting to absolute (0,0), which lands on the non-primary laptop screen) and the fix pattern, demonstrated on IntelliJ IDEA. Reuse this diagnosis for other apps with the same symptom before assuming it's a GNOME/Mutter setting.
version: 1.0.0
allowed-tools: Bash, Read, Edit
---

# App opens on wrong monitor (uzer-Latitude-5420, dual monitor)

## Monitor layout on this machine

From `~/.config/monitors.xml`:

- `eDP-1` (laptop panel): logical position `x=0, y=0`, 1920x1080. **Not** primary.
- `DP-1` (BenQ PD2705Q, external): logical position `x=1920, y=0`, 2560x1440. **Primary**
  (`<primary>yes</primary>`).

So the primary monitor is offset to the right, and the origin `(0,0)` of the
virtual desktop sits on the laptop panel — a non-primary screen. This matters
because many apps fall back to coordinate `(0,0)` when they have no saved
window position, which puts them on the *wrong* screen on this specific setup.

Re-check this file if the physical monitor arrangement changes (different
dock, different external monitor) — the primary's offset may no longer be
`x=1920`.

## Root cause (confirmed, not a guess)

This is **not** Mutter/GNOME remembering a "last used monitor" and not a
cursor-follows-focus placement heuristic — there is no such per-app or
per-window memory at the compositor level on Wayland (clients can't request
absolute screen coordinates, and Mutter doesn't persist monitor choice across
app launches). Confirmed by testing: deleting an app's own saved frame
position still placed it at `(0,0)` — i.e. the fallback is the *app's own*
default, not something Mutter re-derives.

The actual cause is each app/toolkit persisting **its own** window geometry,
and defaulting to `(0,0)` (or whatever it last saved there) when uncertain.
Where that state lives depends on the toolkit: GTK → dconf, Qt →
`~/.config/<app>/*.conf`, Electron → its own JSON state file, Java/Swing →
its own XML (documented below for IntelliJ).

## Fix pattern (confirmed on IntelliJ IDEA, Java/Swing)

IntelliJ (JetBrains, Java/Swing/AWT) stores window geometry in:

```
~/.config/JetBrains/<ProductVersion>/options/window.state.xml
```

under:

```xml
<component name="WindowManager">
  <frame x="..." y="..." width="..." height="..." extendedState="6" />
</component>
```

`extendedState="6"` = maximized (Java `Frame` state bitmask:
`MAXIMIZED_HORIZ=2 | MAXIMIZED_VERT=4`).

**Steps:**

1. Make sure the app is fully closed first — it only writes this file on
   clean exit, so editing while running gets silently overwritten.
2. Back up the file.
3. Either delete the `<frame .../>` line (lets the app re-pick a default —
   on this machine that default is `(0,0)`, i.e. the *wrong* screen, so this
   alone does **not** fix it — it just proved the root-cause theory above)
   or, better, **set `x`/`y` explicitly to a coordinate inside the primary
   monitor's bounds** (here: `x ≥ 1920`).
4. Relaunch and confirm placement; the app will re-save its own state from
   there on next clean exit.

Confirmed working value for IntelliJ on this machine:
`<frame x="2475" y="40" width="1440" height="810" extendedState="6" />`.

## Applying this to other software

Same diagnosis likely applies to **any** app that "remembers" its position
and opens on the laptop panel instead of the BenQ. General procedure:

1. Identify the toolkit (Java/Swing, Qt, Electron, GTK) — tells you where the
   state file lives.
2. Find the saved x/y (grep for the app name under `~/.config/`,
   `~/.local/share/`, or check dconf with
   `dconf dump /org/gnome/ | grep -B2 -A2 window`).
3. Set x to `≥ 1920` (inside the BenQ's logical bounds) rather than deleting
   the entry outright — deleting just re-triggers the `(0,0)` fallback and
   lands back on the laptop screen.
4. Close the app fully before editing, confirm after relaunch.

If no per-app state file can be found/edited (e.g. the app doesn't persist
position at all), fall back to the GNOME-level workaround: install the
**Auto Move Windows** extension, set GNOME Settings → Multitasking →
Workspaces to "Workspaces on primary display only", and add a rule pinning
that app's `.desktop` id to workspace 1 — this forces it onto the primary
monitor regardless of its own placement logic.

## Usage history

- **2026-10-18**: IntelliJ IDEA (`IntelliJIdea2025.3`) opening on laptop panel
  instead of BenQ external. Diagnosed via `window.state.xml` +
  `~/.config/monitors.xml`; disproved the Mutter-placement theory by removing
  the saved frame and observing it still fell back to `(0,0)` (laptop panel).
  Fixed by setting an explicit `x/y` inside the BenQ's bounds.
