---
name: fix-desktop-thumbnail
description: Use when Nautilus (GNOME Files) on uzer-Latitude-5420 (Ubuntu 24.04, GNOME Shell 46) isn't showing thumbnails for images or documents — generic icons instead of previews. Covers the confirmed root cause for images (a missing AppArmor conffile for Nautilus after an apparmor package upgrade) and the still-open gap for document/ebook thumbnails (pdf, epub, azw3, kfx).
version: 1.0.0
allowed-tools: Bash, Read
---

# Nautilus / GNOME thumbnails not showing (uzer-Latitude-5420)

Machine: Dell Latitude 5420, Ubuntu 24.04 LTS ("noble"), GNOME Shell 46,
native (non-sandboxed) deb Nautilus. See also **fix-desktop**
(`~/.claude/skills/fix-desktop/`) for the general log of small desktop fixes
on this machine — this one got its own skill because the diagnosis had
multiple dead ends and a non-obvious root cause worth preserving in full.

## Symptom

Files appear as generic icons in Nautilus instead of image/document
thumbnails.

## Confirmed root cause (images) — missing AppArmor profile for Nautilus

An `apparmor` package upgrade (`24.04.7 → 24.04.8`) left a **missing
conffile**: `/etc/apparmor.d/nautilus`. Without it, Nautilus had no loaded
AppArmor profile (`aa-status`), which broke its ability to spawn thumbnailer
helpers for images.

Diagnose:

```bash
dpkg -V apparmor                      # reports missing conffile if this is the cause
aa-status | grep -i nautilus          # profile absent if broken
cat /etc/apt/history.log* | grep -i apparmor   # check for a recent apparmor upgrade
```

Fix:

```bash
sudo apt-get install -y --only-upgrade apparmor libapparmor1 bubblewrap
sudo apt-get install --reinstall -o Dpkg::Options::="--force-confmiss" apparmor
```

This restores `/etc/apparmor.d/nautilus` (an `unconfined`-label profile that
just gives Nautilus a profile name). Confirm with `dpkg -V apparmor` (clean)
and that image thumbnails reappear.

### Dead ends ruled out while diagnosing (images)

Checked and NOT the cause — skip re-checking these unless something changed:

- thumbnailer gsettings (`org.gnome.desktop.thumbnailers disable-all`), tumbler
- installed thumbnailer packages (`dpkg -l | grep -iE 'thumbnail|ffmpegthumbnailer|raw-thumbnailer|libgsf|poppler'`), `/usr/share/thumbnailers/`
- Nautilus version, `gio mime` for jpg/png
- `~/.cache/thumbnails/fail/gnome-thumbnail-factory/` Thumb::URI inspection (useful for *noticing* the pattern, not the cause)
- manually running `gdk-pixbuf-thumbnailer` on a real PNG/JPG (worked standalone — proved the thumbnailer binary itself was fine)
- `org.gnome.desktop.thumbnail-cache maximum-size`, `org.gnome.nautilus.preferences thumbnail-limit`
- `gdk-pixbuf-query-loaders` cache
- snap/flatpak sandboxing — Nautilus here is native deb, not sandboxed
- user journal grep for thumbnail errors; `eog` + journal; libgnome-desktop thumbnail libs
- clearing `~/.cache/thumbnails/fail`, restarting `nautilus`, disk space, cache dir permissions
- dbus `org.freedesktop.thumbnails.Thumbnailer1` monitoring, kernel/audit log AppArmor denials/seccomp — this is what eventually pointed at AppArmor

## Fix: another user account — old files still show generic icons, only new files thumbnail

The AppArmor fix above is system-wide (one `/etc/apparmor.d/nautilus`
profile for the machine), so it covers every user account. But if a
secondary account (e.g. `hop`, reachable via `sudo su -l hop`) hit the bug
*before* the fix was applied, its old failures are stuck forever: GNOME's
`~/.cache/thumbnails/fail/gnome-thumbnail-factory/` is a **permanent
negative cache keyed by file URI + mtime** — once an attempt fails for a
given file, Nautilus never retries it unless the file's mtime changes. New
files (never attempted before) thumbnail fine because they've never been
cached as failed; old files stay stuck as generic icons forever even though
generation would now succeed.

Confirm this is the situation (not a fresh AppArmor gap for that user):

```bash
sudo aa-status | grep -i nautilus     # profile present → fix already applied machine-wide
sudo dpkg -V apparmor                 # clean → not a fresh apparmor regression
sudo su -l <user> -c 'ls ~/.cache/thumbnails/fail/gnome-thumbnail-factory/ | wc -l'
```

Fix — just clear that user's stale fail cache, it will repopulate correctly:

```bash
sudo su -l <user> -c 'rm -rf ~/.cache/thumbnails/fail/gnome-thumbnail-factory'
```

Verify by manually thumbnailing one of the previously-failing files as that
user (`gdk-pixbuf-thumbnailer <file> /tmp/test.png`, expect exit 0); Nautilus
will regenerate real thumbnails for old files next time it browses those
folders.

## Open problem (not yet fixed): document/ebook thumbnails

After the AppArmor fix, `~/.cache/thumbnails/fail/gnome-thumbnail-factory/`
kept accumulating failures — but only for **document/ebook formats**, not
images:

- `pdf`
- `epub`
- `azw3` (Kindle)
- `kfx` (Kindle)
- occasional stray `png` (not yet explained)

Likely cause: no thumbnailer is installed/registered for those mime types
(candidates: `ffmpegthumbnailer`, an `evince`/poppler PDF thumbnailer, an
ebook thumbnailer). Not yet investigated — if picking this up, start with
`/usr/share/thumbnailers/*.thumbnailer` to see what's registered per mimetype
and `dpkg -l | grep -iE 'poppler|evince|ebook'`.

## Usage history

- **2026-09-29 → 2026-10-02**: image thumbnails broken, root-caused to
  missing `/etc/apparmor.d/nautilus` conffile after an apparmor package
  upgrade, fixed by reinstalling apparmor with `--force-confmiss`.
- **2026-10-07 → 2026-10-23 (ongoing)**: document/ebook thumbnails (pdf,
  epub, azw3, kfx) still failing — separate, unfixed issue.
- **2026-11-06**: secondary account `hop` — old files stuck as generic icons,
  new files thumbnailed fine. AppArmor profile was already correct
  machine-wide; cause was 272 stale entries in `hop`'s own
  `~/.cache/thumbnails/fail/gnome-thumbnail-factory/` predating the fix
  (GNOME's fail cache never retries a URI+mtime it already marked failed).
  Fixed by clearing that user's fail-cache directory.
