# ADR-003: Digest is one .m4a per morning with embedded MP4 chapters, assembled by system ffmpeg

- Status: Accepted (2026-07-11)
- Deciders: product owner (episode shape P6), Fable (container/tooling detail)

## Context

Owner chose "one digest file per morning with per-article chapter markers" delivered as plain local files (no RSS hosting). The file must play well in the Apple ecosystem (iPhone Files app, QuickTime, Apple Books/Music import) and expose chapters where supported.

## Decision

1. Output container: **`.m4a` (MP4/AAC-LC)**, mono, 24 kHz, 64 kbps, loudness-normalized to −16 LUFS (podcast speech convention).
2. Chapters: **MP4 chapter atoms written by ffmpeg** from an FFMETADATA1 file (`TIMEBASE=1/1000`, one `[CHAPTER]` per article + opening/closing), titles = article titles.
3. Assembly tool: **system ffmpeg/ffprobe** (Homebrew), invoked via `execFile` with generated silence WAVs between segments/chapters; earmark bundles no audio binaries.
4. Naming: `earmark-YYYY-MM-DD.m4a` (ASCII, sortable), one per local date (sequence suffix only with `--force`).

## Consequences

- Chapter visibility varies by player (QuickTime/Apple Books show them; iOS Files preview may not) → validation criterion is `ffprobe -show_chapters` correctness plus a documented manual playback check; a `.m4b` rename variant for Apple Books is a recorded v2 idea.
- ffmpeg becomes a hard runtime prerequisite → checked by `earmark doctor`, documented in SETUP.
- WAV intermediates (24 kHz/16-bit mono) are the fixed inter-provider contract, keeping concat trivial and deterministic.

## Alternatives considered

- **MP3 + ID3v2 CHAP frames**: broader legacy support but chapter tooling is weaker in ffmpeg and Apple players prefer MP4 chapters — rejected.
- **One file per article**: contradicts confirmed P6 — rejected (recorded as v2 optional mode).
- **Node audio libraries instead of ffmpeg**: no maintained pure-JS AAC encoder with chapters; ffmpeg is ubiquitous and scriptable — rejected.
- **Opus/.ogg**: poor Apple-native support — rejected.
