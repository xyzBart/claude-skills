---
name: ocr
description: Full workflow to OCR phone-photographed paper documents - cleans up shadows/uneven lighting/tinted background into a crisp black-and-white image, then runs tesseract to produce a searchable PDF and a plain-text file. Use when asked to OCR scans/photos of documents, fix scan background before printing, or produce searchable PDF + txt from photos of paper pages. Defaults to Polish (-l pol) but language is a parameter.
version: 1.1.0
allowed-tools: Bash
---

# OCR workflow: scan cleanup + tesseract

Turns a phone photo of a paper document into three outputs sharing one
basename:

1. `<base>_bw.jpg` — cleaned black-on-white image (no shadow/background tint)
2. `<base>_bw.pdf` — searchable PDF (tesseract's invisible-text-over-image PDF, built from the cleaned image)
3. `<base>_bw.txt` — plain-text OCR result

Always produce all three, never just the PDF or just the text — the image is
the visual proof of what OCR saw, and the two text formats serve different
downstream uses (txt for grepping/diffing, pdf for reading/printing/archiving).

Never overwrite the original source photo or any pre-existing output — this
pipeline is lossy (binarization discards color/shading permanently), so it
always writes under a new basename (the `_bw` suffix) rather than in place.

## Why binarize before OCR at all

Tesseract's `pdf` output format always embeds the *input image* as the page
background with an invisible text layer on top — there is no flag to drop
the image and keep only text-on-white. If the input is a raw phone photo,
the background is grayscale/color with shadows and uneven lighting, which
prints as a dark, ink-heavy page even though it "looks fine" on screen.
Thresholding the image to pure black-and-white *before* running tesseract
fixes this: the embedded background becomes true white, so it prints clean.

As a bonus this usually also improves OCR accuracy, not just print quality —
in practice it fixed several misreads caused by the model treating shadow
noise as characters (e.g. a phone number misread as `78 200` from a shadowed
photo came back correctly as `73 200` after cleanup).

## Step 1 — check the tools are present

```bash
which convert tesseract
tesseract --list-langs   # confirm the language pack you need is listed (e.g. "pol")
```

If the language pack is missing, stop and tell the user which package to
install (e.g. `tesseract-ocr-pol` on Debian/Ubuntu) rather than silently
falling back to a different language.

## Step 1.5 — check orientation before binarizing

Phone photos often carry rotation as EXIF metadata (`Orientation` tag) rather
than actually rotating the pixels. Image viewers apply that tag
automatically, so the photo "looks right" when you open it — but tools like
`convert`/`tesseract` that read raw pixel data will not, and will silently
process it sideways.

```bash
identify -format "%[EXIF:Orientation]\n" "$SRC.jpg"   # anything other than "1" (or empty) means rotated
tesseract "$SRC.jpg" - --psm 0 2>&1 | grep -E "Orientation in degrees|Rotate"
```

Fix is to always include `-auto-orient` as the *first* operation in the
`convert` pipeline (before `-colorspace Gray` etc.) — it reads the EXIF tag
and physically rotates the pixels to match, then the tag is cleared so
nothing downstream double-rotates it. Do this unconditionally; it's a no-op
on images that don't need it, so there's no reason to special-case it.

## Step 2 — binarize the image (the part that took tuning)

The working command:

```bash
convert "$SRC.jpg" -auto-orient -colorspace Gray -negate -lat 40x40+15% -negate "$SRC_bw.jpg"
```

`-lat WxH+N%` is *local* adaptive thresholding: it computes a threshold per
neighborhood instead of one global value, which is what actually copes with
a phone photo's uneven lighting/shadow gradient across the page.

### What was tried and didn't work, and why

- **Global threshold** (`convert -colorspace Gray -threshold 60%`): looks
  fine where lighting is even, but any shadowed corner/edge of a phone photo
  falls below the single global cutoff and comes out as solid black speckle
  noise. Adding `-despeckle -despeckle` before the threshold reduced the
  speckling a little but didn't remove it — the underlying problem is a
  brightness *gradient*, not isolated noise pixels, so a spatial filter
  can't fix what a non-adaptive threshold gets wrong.
- **`-lat` without the `-negate … -negate` wrap** (i.e. `-lat 40x40+15%`
  directly on the grayscale image): inverted the polarity — output was
  white text on a black page instead of black text on white. Wrapping it in
  `-negate` before and after keeps the polarity correct while still getting
  adaptive behavior; the plain form is not reliable across document photos.
- **Small window** (`-lat 25x25+10%`): too small a neighborhood to average
  out shadow gradients over a full page — same inverted/patchy problems as
  above, just worse. `40x40+15%` was the size that held up across a batch of
  15 varied phone-photo pages; if a new batch still shows patchy or inverted
  results, try a bigger window (e.g. `60x60`) before reaching for something
  else.

If you're processing a new batch and the default doesn't look clean, test on
one page first (render it and look at it) before running the whole batch —
tune the window size/offset, don't switch techniques speculatively.

## Step 3 — OCR into PDF + TXT from the cleaned image

```bash
tesseract "$SRC_bw.jpg" "$SRC_bw" -l pol txt pdf
```

This produces `$SRC_bw.pdf` and `$SRC_bw.txt` in one invocation (tesseract
takes the output *basename*, not a file extension, and writes one file per
requested format). Swap `-l pol` for the document's actual language, or e.g.
`-l pol+eng` for mixed-language pages.

## Batch processing

```bash
for f in *.jpg; do
  base="${f%.jpg}"
  [[ "$base" == *_bw ]] && continue   # don't reprocess already-cleaned output
  convert "$f" -auto-orient -colorspace Gray -negate -lat 40x40+15% -negate "${base}_bw.jpg"
  tesseract "${base}_bw.jpg" "${base}_bw" -l pol txt pdf
done
```

## Step 4 — verify before reporting done

Spot-check at least one page visually before telling the user it's done:

```bash
pdftoppm -jpeg -r 150 "$SRC_bw.pdf" /path/to/scratch/preview
```

Read the rendered preview image and confirm: white background (no gray
tint/speckle), text is crisp and correctly oriented. Also skim the `.txt`
output for obvious garbage (e.g. a logo/crest OCR'd as random symbols is
fine to ignore — it's not real text; garbled body text is not).

## Optional — manual vision transcription (only if asked for, on top of OCR)

**This is not part of the default workflow.** Steps 1–4 above (tesseract) are
the deliverable unless the user separately asks for this, e.g. after seeing
the OCR result and asking to double-check it, or asking for a markdown
transcription. Don't reach for it on your own just because a page looks
hard — tesseract's output plus a note about which pages look weak is the
right default outcome.

### When tesseract genuinely can't cope

Tesseract assumes roughly axis-aligned text. Binarization and `-auto-orient`
fix lighting and 90°/180° rotation, but they don't fix **perspective
distortion** — a photo taken at an angle of a bound document (book/binder
photographed from the side, common when someone flips pages one-handed).
The top of the page is further from the camera and compressed diagonally;
tesseract reads this as near-garbage (single stray characters) even though
a human can read it fine once zoomed in. This is the case to reach for
manual transcription — not general OCR sloppiness, but this specific
perspective-skew failure mode.

### The technique

1. Read the full source photo first (not the `_bw` version — the original
   has more tonal detail in the shadows) to get the overall content and
   layout.
2. For anything that must be character-perfect — hashes, ID/PESEL numbers,
   dates, account numbers, commands, URLs — crop and zoom that specific
   region before transcribing it, rather than trusting a read of the whole
   page. Auto-orient once into a temp file, then crop by pixel coordinates
   against *that* oriented copy (cropping before orienting gives you
   coordinates in the wrong frame):
   ```bash
   convert "$SRC.jpg" -auto-orient "$scratch/oriented.jpg"
   convert "$scratch/oriented.jpg" -crop WIDTHxHEIGHT+X+Y +repage -resize 200% "$scratch/zoom.jpg"
   ```
   Read the crop. If a hash/number still isn't unambiguous, crop tighter and
   resize more (300%+) rather than guessing from the wider view.
3. **Don't trust your own first full-page read of dense body text either.**
   A fast read of a whole angled page produces fluent-sounding sentences
   that are sometimes simply wrong — reconstructed from context rather than
   actually read. If a sentence reads as grammatically odd or doesn't quite
   follow, that's a signal to crop-and-zoom that exact line and re-check,
   not to smooth it over. (In practice this caught real errors: a full-page
   read produced "Jak zeznali ustalili..." where the zoomed crop clearly
   showed "Jak żeśmy ustalili...", and invented a plausible-sounding but
   wrong sentence about which bank apps were involved.)
4. Transcribe **verbatim**, including the source's own grammar hiccups,
   typos, and inconsistent phrasing (interrogation protocols transcribe
   dictated speech — awkward phrasing is often genuinely in the original,
   not a reading error). Don't silently "fix" the source. Mark anything you
   truly can't resolve even after zooming as `[nieczytelne]` rather than
   guessing.

### Output convention

One `.md` file per source image, same basename, next to the `_bw.jpg/.pdf/.txt`
outputs (e.g. `20260618_101251.jpg` → `20260618_101251.md`). Structure:

```markdown
# Transkrypcja: <filename>.jpg

> One-line note on photo quality/angle and anything caveated (e.g. "sumy
> kontrolne zweryfikowane na osobnych powiększonych wycinkach — pewność
> wysoka" or "adnotacja odręczna, odczyt przybliżony").

<transcribed content, mirroring the document's structure>
```

Use markdown to mirror the original's structure, not just its words: `**bold**`
for form labels/filled-in values, `☐` for empty checkboxes, `~~strikethrough~~`
for crossed-out template options (e.g. `świadek/~~biegły~~*`), *italics* for
file/app names or handwritten annotations, and `` `code spans` `` for
hashes/commands/identifiers that need to visually stand out as literal data.
