---
name: chromium-screenshot
description: Take a headless-Chromium screenshot of a local HTML file on this machine (e.g. to visually verify a generated report/page). Use when asked to screenshot, preview, or visually check a local HTML file with chromium/chromium-browser.
version: 1.0.0
allowed-tools: Bash
---

# Headless Chromium screenshot on this machine

Both `chromium` and `chromium-browser` on this machine resolve to the **snap**
build of Chromium (`/usr/bin/chromium-browser` is just a wrapper script that
execs `/snap/bin/chromium`). There is only one real binary.

## The gotcha: snap gives Chromium a private /tmp

Snap confinement puts Chromium in its own mount namespace with a **private
`/tmp`** (like systemd's `PrivateTmp`). This means:

- A source HTML file living under `/tmp/...` (including any scratchpad
  directory under `/tmp/...`) is invisible to Chromium — loading it produces
  `ERR_FILE_NOT_FOUND` even though the file exists on the host.
- Writing `--screenshot=/tmp/...png` fails with
  `Failed to write file: ... No such file or directory`, again even though
  the target directory exists on the host.

This is **not** the same thing as Chromium's internal renderer sandbox
(`--no-sandbox` does not fix it) and it is **not** a permission you can grant
with `snap connect` — there is no snap interface for shared `/tmp` access.
Check `snap connections chromium` if you want to confirm; you'll see `home`
and `removable-media` connected, but nothing exposing `/tmp`.

## Fix: stage files under $HOME first

The `home` interface is already connected, so `$HOME` (and its subdirs) is
visible to the snap. Copy the HTML in and screenshot out through `$HOME`
instead of `/tmp`:

```bash
cp /tmp/.../report.html ~/preview.html

chromium-browser --headless --disable-gpu --no-sandbox \
  --screenshot=$HOME/preview.png --window-size=1400,1000 \
  "file://$HOME/preview.html"

# view $HOME/preview.png, then clean up:
rm -f ~/preview.html ~/preview.png
```

`--no-sandbox` is still worth keeping (avoids unrelated Chromium
sandbox/SUID issues in this environment), but it does nothing for the
snap-private-`/tmp` problem — the `$HOME` staging step is what actually
fixes it.

## The gotcha: --window-size below ~500px wide is silently ignored

`--headless --screenshot=out.png --window-size=W,H` for `W` below about 500
does **not** actually lay the page out at `W` pixels wide — Chromium's
headless screenshot mode clamps the internal layout viewport to a ~500px
floor regardless of a smaller requested width, confirmed via
`window.innerWidth` injected into the page. But the *output PNG* is still
cropped to the requested `W×H`. Net effect: you get a screenshot that looks
like the page overflows/gets cut off at the right edge (e.g. text appearing
truncated), when really the page laid out fine at 500px and the image is
just a crop of the left `W` pixels of that wider layout. This reproduces
with `--headless=old` too and isn't fixed by `--force-device-scale-factor=1`.

This matters for verifying phone-width (~360-430px) layouts: a screenshot at
`--window-size=390,844` is **not trustworthy** for judging overflow/wrapping
— you're actually looking at a crop of a 500px-wide render, not a true
390px one.

Workarounds, in order of preference:
1. **Verify via injected JS instead of pixels** for anything overflow/width
   sensitive: load the page and check `el.scrollWidth <= el.clientWidth`
   (no overflow) rather than eyeballing a cropped screenshot.
2. **Screenshot at ≥500px** (e.g. 600) as an approximate "narrow viewport"
   stand-in when a true phone-width visual is only for a sanity check, not
   a precise layout judgment.
3. Standard CSS (percentage widths, default `white-space: normal` wrapping)
   that already works at ~484px content width via method 1 will wrap
   correctly on an even narrower real phone too — don't chase pixel-exact
   confirmation below the 500px floor, it isn't obtainable this way.

## Don't reach for these

- `snap install chromium --devmode` / switching to `classic` confinement —
  removes Chromium's own sandboxing (renderer isolation for untrusted web
  content, not just the `/tmp` restriction). Real security downgrade for a
  browser; don't do this just to avoid a `cp` step.
- Installing a non-snap Chromium/Chrome — possible in principle, but heavier
  than the problem warrants; the `$HOME` staging workaround is enough.
