# macOS `say` fallback TTS provider (en)

## Summary

Implement `src/tts/say.ts`: a zero-install English `TtsProvider` shelling out to `/usr/bin/say` (text passed via a temp file, never argv/shell), converting the AIFF output to the contract WAV with ffmpeg.

## Context

Insurance for U1 (kokoro-js may not load on some Node/macOS combos) and for users who refuse the model download. Selected via `tts.en.provider: "say"`. Quality is accepted as inferior (research/local-tts-selection.md).

## Scope

- `src/tts/say.ts`, tests (arg construction unit tests; darwin-only integration). No new runtime deps.

## Detailed Requirements

1. Constructor `new SayProvider(cfg.tts.en.say, ffmpegPath, cacheDir, logger)`; `id='say'`, `lang='en'`.
2. `checkAvailability()`: `/usr/bin/say` exists+executable AND configured voice appears in `execFile('/usr/bin/say', ['-v','?'])` output (line-prefix match, case-sensitive) → ok; missing voice → `{ok:false, detail:"voice <v> not installed — System Settings → Accessibility → Spoken Content, or pick another (say -v '?')"}`; non-darwin platform → `{ok:false, detail:"say provider is macOS-only"}`.
3. `synthesize(utterance, outWavPath)`:
   - write utterance text to `<workdir-temp>.txt` (UTF-8) — **text never appears in argv** (immune to `-`-prefixed text and length limits; DESIGN §10.5)
   - `execFile('/usr/bin/say', ['-v', cfg.voice, '-r', String(cfg.wordsPerMinute), '-o', tmpAiff, '-f', txtPath])`, timeout 120 s
   - convert: `execFile(ffmpegPath, ['-y','-hide_banner','-loglevel','error','-i', tmpAiff, '-ar','24000','-ac','1','-c:a','pcm_s16le', outWavPath])`
   - duration via issue 16 `wavDurationMs`; cleanup temp txt/aiff in `finally`
   - failures (non-zero exit, timeout) → `TTS_SYNTH_FAILED` with stderr excerpt (≤ 200 chars, control-stripped); no retry (deterministic local tool).
4. Both child invocations: `execFile` with argument arrays, no `shell:true`, no string interpolation (boundary B6).
5. Empty/whitespace-only utterance text → skip synthesis, error `TTS_SYNTH_FAILED` "empty utterance" (upstream splitter should never produce it; fail loudly if it does).

## Acceptance Criteria

- [ ] Arg-construction unit tests (execFile injected/spied): exact argv arrays for say and ffmpeg incl. text starting with `-rate`, containing `"; rm -rf ~"`, newlines, and 5,000 chars — text only ever lands in the temp file (spy asserts argv never contains it).
- [ ] Temp files removed on success and on ffmpeg failure.
- [ ] `checkAvailability` matrix: ok / missing voice / non-darwin (mock `process.platform` via injected platform getter).
- [ ] Darwin integration test (CI macos runner: real say + real ffmpeg): synthesize "Good morning." → WAV at 24 kHz mono 16-bit, duration 300–3000 ms; runs in CI (not gated) since both tools exist on `macos-14`.

## Validation

`vitest` (integration included in CI). Manual: none beyond listening spot-check at wave-4 gate if `say` is the active provider.

## Dependencies

16 (+ ffmpeg path from 02).

## Non-goals

Japanese via say; voice installation automation; quality tuning beyond rate; making say the default (kokoro remains default unless the owner decides otherwise after U1 evidence).

## Design References

DESIGN §10.5, §13.1 B6, §9.3; research/local-tts-selection.md; ISSUE_PLAN U1.
