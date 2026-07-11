# Audio assembly: ffmpeg concat, loudnorm, m4a chapters, tags

## Summary

Implement `src/audio/assemble.ts` (+ `src/audio/ffprobe.ts`): take the ordered synthesized WAV segments with chapter/paragraph structure, interleave silence gaps, compute chapter timestamps, and produce the final tagged, loudness-normalized `.m4a` with embedded MP4 chapters via system ffmpeg — atomically placed in the output directory.

## Context

This is the last pipeline stage and the product's deliverable (ADR-003). Chapter math must be exact (players navigate by it), metadata must be escaped correctly (titles are attacker-influenceable article titles — boundary B6/B7), and output placement must never leave a half-written file where iCloud could sync it.

## Scope

- `src/audio/assemble.ts`, `src/audio/ffprobe.ts`, tests (real ffmpeg on CI). No new runtime deps (ffmpeg is a system binary).

## Detailed Requirements

1. Input model:
   ```ts
   AssembleInput = {
     date: string; sequence: number;             // naming
     chapters: Array<{ title: string; wavs: Array<{ path: string; durationMs: number; paragraphBreakAfter: boolean }> }>;
     cfg: { gapBetweenChaptersMs, gapBetweenParagraphsMs, fileNamePrefix, outputDir, ffmpegPath, ffprobePath, workDir, outputLanguage };
   }
   AssembleResult = { outputPath, durationMs, chapterTimes: Array<{title, startMs, endMs}> }
   ```
2. Plan construction (pure function `planConcat(input)` exported for tests):
   - sequence: for each chapter c: (c > 0 → chapter-gap silence) then its wavs, inserting paragraph-gap silence **after** any wav with `paragraphBreakAfter` except the chapter's last wav
   - silence files: generated into workDir via issue 16 `writeSilenceWav` — one file per distinct duration, reused
   - chapter `startMs` = cumulative sum at the chapter's first element (the chapter gap **precedes** the chapter and belongs to the previous chapter's end — i.e. chapter N starts after the gap); `endMs` = start of next chapter's gap or total; all integer ms
   - total planned duration = Σ segment durations + gaps.
3. FFMETADATA1 file (`writeFfmetadata(chapterTimes, tags)`):
   - header `;FFMETADATA1`; global tags: `title=earmark <localized date>` (ja: `earmark 2026年7月11日`; en: `earmark July 11, 2026`), `album=earmark`, `artist=earmark`, `genre=Podcast`, `date=YYYY-MM-DD`
   - per chapter: `[CHAPTER]\nTIMEBASE=1/1000\nSTART=<ms>\nEND=<ms>\ntitle=<escaped>`
   - escaping per ffmpeg metadata spec: backslash-escape `=`, `;`, `#`, `\` and newline in values; titles also control-stripped and ≤ 250 chars.
4. Concat list file: ffconcat format `ffconcat version 1.0` + `file '<path>'` lines with single-quote escaping (`'` → `'\''`); paths are workDir-internal (ULID-named) but escape anyway.
5. ffmpeg invocation (single pass; exact argv, execFile; timeout via injected option, default 10 min — see req 10):
   ```
   <ffmpeg> -hide_banner -loglevel error -y \
     -f concat -safe 0 -i <list.txt> \
     -f ffmetadata -i <meta.txt> \
     -map_metadata 1 -map_chapters 1 -map 0:a \
     -af loudnorm=I=-16:TP=-1.5:LRA=11 \
     -c:a aac -b:a 64k -ar 24000 -ac 1 \
     -movflags +faststart \
     <workDir>/digest.m4a
   ```
   (`-f ffmetadata` forces the metadata demuxer regardless of extension; `-map_chapters 1` maps chapters explicitly — `-map_metadata` alone does not carry chapters reliably.) **Implementation checkpoint**: verify chapters survive this exact pipeline with the installed ffmpeg; if the single pass still drops chapters, fall back to two passes (encode → `-i digest.m4a -f ffmetadata -i meta.txt -map_metadata 1 -map_chapters 1 -c copy remux.m4a`) and record which path was needed in a code comment + PR (U5-adjacent evidence).
6. Verification before publish (`ffprobe.ts`): `execFile(ffprobe, ['-v','error','-print_format','json','-show_chapters','-show_format', file])` → assert: format duration within ±2% of plan, chapter count/titles/START order exactly match plan (±200 ms per DESIGN §9.4). Mismatch → `EarmarkError ASSEMBLE_FAILED` (workDir kept for debugging).
7. Atomic publish: target name `<prefix>-<date>.m4a` (sequence ≥ 2 → `<prefix>-<date>-<seq>.m4a`); copy `digest.m4a` to `<outputDir>/.<target>.part` then `fs.rename` to final (same volume by construction); ensure outputDir exists; pre-existing final file without `--force` semantics is the caller's concern (orchestrator checks idempotency before assembling — this module overwrites `.part` freely, never overwrites a final file: existing final → `ASSEMBLE_FAILED` "target exists").
8. workDir cleanup on success (delete run's work directory); on failure keep + prune-old-workdirs helper `pruneWorkDirs(cacheDir, olderThanDays=7)` exported for the orchestrator.
9. Failure mapping: ffmpeg non-zero → `ASSEMBLE_FAILED` with last 500 chars of stderr (control-stripped via issue 04 `stripControl`).
10. Options parameter for testability: `assemble(input, opts?: { execTimeoutMs?: number /* default 600_000 */ })` — applied to both ffmpeg and ffprobe invocations.

## Acceptance Criteria

- [ ] `planConcat` unit tables: 2 chapters × (2 wavs w/ paragraph break + 1 without) with gaps 1500/500 → exact expected element sequence and chapter times (hand-computed values in the test).
- [ ] FFMETADATA escaping: title `a=b;c#d\e\nf` roundtrips through ffprobe chapter title byte-exact (integration assert).
- [ ] Chapter integration (real ffmpeg): 3 chapters of test WAVs → m4a exists; ffprobe chapters count 3, titles match (incl. ja title `『テスト』`), starts within ±200 ms of plan, duration within ±2%.
- [ ] Tag assertions exact: `title` = `earmark 2026年7月11日` for ja (and one en case `earmark July 11, 2026` via a formatter unit test), `album=earmark`, `artist=earmark`, `genre=Podcast`, `date=2026-07-11` (ffprobe format tags).
- [ ] Loudnorm smoke on a **non-silent** fixture (440 Hz tone WAVs generated in-test — silence would measure −inf LUFS): output integrated loudness −16 ±1.5 LUFS via a `loudnorm=print_format=json` measurement pass on the OUTPUT file.
- [ ] Cleanup semantics: success deletes the run's work dir; injected ffprobe-mismatch failure keeps it; `pruneWorkDirs(cacheDir, 7)` deletes only earmark work dirs older than 7 days (fixture mtimes) and never a passed-in current dir.
- [ ] Existing final file → `ASSEMBLE_FAILED` without touching it; `.part` file never left behind on failure (assert dir listing).
- [ ] `execTimeoutMs` option enforced (tiny timeout + injected slow stub → `ASSEMBLE_FAILED`).

## Validation

`vitest` integration on macOS CI (ffmpeg present via brew in workflow — extend the CI workflow in this PR if issue-01's workflow lacks ffmpeg installation; document in PR). Manual: assemble a real 2-article digest at wave-4 gate; open in QuickTime + Apple Books; screenshot chapter list → attach (U5 evidence).

## Dependencies

02, 16, 21 (chapter/paragraph structure semantics).

## Non-goals

Cover art (v1 non-goal); `.m4b` variant (v2); multi-bitrate; parallel encoding; talking to the DB (orchestrator records results).

## Design References

DESIGN §9.4, §13.1 B6/B7, §12.2; ADR-003; ISSUE_PLAN U5.
