---
name: youtube-summarize
description: Use this skill when the user provides a YouTube URL and wants it summarized, transcribed, or analyzed. Triggers on any YouTube link (youtube.com, youtu.be) combined with words like summarize, summary, transcript, watch, explain, or tldr.
version: 1.0.0
allowed-tools: Bash, Read
---

# YouTube Summarize Skill

## Workflow

### 1. Download subtitles

Run yt-dlp to fetch auto-generated or manual subtitles in VTT format, writing to a temp file:

```bash
TMPFILE=$(mktemp /tmp/yt-sub-XXXXXX)
~/.local/bin/yt-dlp \
  --skip-download \
  --write-auto-sub \
  --write-sub \
  --sub-lang en \
  --sub-format vtt \
  --convert-subs vtt \
  -o "$TMPFILE" \
  "<URL>" 2>&1
# Subtitle file will be at ${TMPFILE}.en.vtt (yt-dlp appends language/extension)
```

- If subtitles are available the file `${TMPFILE}.en.vtt` (or similar) will be created.
- If no English subtitles exist, retry without `--sub-lang en` to get any available language, then note the language to the user.
- If yt-dlp reports no subtitles at all, tell the user and stop.

### 2. Clean the VTT

Strip VTT header, timestamps, and duplicate lines to get plain text:

```bash
grep -v '^WEBVTT' "${TMPFILE}.en.vtt" \
  | grep -v '^[0-9]' \
  | grep -v '^-->' \
  | grep -v '^\s*$' \
  | awk '!seen[$0]++' \
  > "${TMPFILE}.txt"
```

Then read `${TMPFILE}.txt` with the Read tool.

### 3. Clean up temp files

```bash
rm -f "$TMPFILE" "${TMPFILE}".*.vtt "${TMPFILE}.txt"
```

### 4. Summarize

Write a structured summary with these sections (omit sections that aren't applicable):

- **Title / Channel** — from yt-dlp output or the transcript itself
- **TL;DR** — 2-3 sentence gist
- **Key Points** — bulleted list of the main ideas
- **Notable Details** — interesting specifics, quotes, data points
- **Conclusion** — how the video wraps up

Tailor length to the video: a 3-minute clip gets a short summary; a 2-hour lecture gets a detailed one.

## Notes

- yt-dlp binary: `~/.local/bin/yt-dlp`
- Subtitles are preferred over audio transcription (faster, no model needed).
- If the user asks for a full transcript instead of a summary, output the cleaned plain text directly.
- If the user asks for specific information from the video, answer the question using the transcript as context.
