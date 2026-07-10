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

## Don't reach for these

- `snap install chromium --devmode` / switching to `classic` confinement —
  removes Chromium's own sandboxing (renderer isolation for untrusted web
  content, not just the `/tmp` restriction). Real security downgrade for a
  browser; don't do this just to avoid a `cp` step.
- Installing a non-snap Chromium/Chrome — possible in principle, but heavier
  than the problem warrants; the `$HOME` staging workaround is enough.
