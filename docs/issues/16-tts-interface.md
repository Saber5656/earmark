# TTS provider interface, utterance splitter, WAV contract

## Summary

Implement `src/tts/types.ts` (the `TtsProvider` interface and utterance model), the shared utterance splitter (sentence-merging with language-specific target limits), and `src/audio/wav.ts`: dependency-free silence-WAV generation, PCM16 WAV writing, and a robust WAV header/duration reader. These fix the audio contract every provider and the assembler rely on.

## Context

ADR-004 keeps providers thin; sizing text into engine-friendly utterances and producing standard intermediate audio (24 kHz/16-bit/mono PCM WAV — DESIGN §9.3) is shared logic. Silence generation feeds paragraph/chapter gaps (DESIGN §9.4) without invoking ffmpeg per gap. `writePcm16Wav` is consumed by the Kokoro provider (issue 19).

## Scope

- `src/tts/types.ts`, `src/tts/utterance.ts`, `src/audio/wav.ts`, tests. No new runtime deps.

## Detailed Requirements

1. `tts/types.ts` per DESIGN §10.1: `TtsUtterance = {text, index}`; `TtsProvider` with `id`, `lang`, `checkAvailability(): Promise<ProviderHealth>` (type from `core/provider.ts`, issue 14), optional `prepare()`, `synthesize(u, outWavPath) → Promise<{durationMs}>`, optional `dispose()`.
2. Utterance splitter — exact signature:
   ```ts
   type UtterancePlan = Array<{ text: string; paragraphBreakAfter: boolean }>;
   function splitUtterances(paragraphs: string[], lang: 'ja'|'en'): { plan: UtterancePlan; hardSplits: number };
   ```
   (both types exported from `src/tts/utterance.ts`)
   - **target merge limits**: ja 120 chars, en 280 chars — sentences (from `splitSentences` in `core/textseg.ts`, issue 12) are greedily merged (joined with a single space for en, no separator for ja) while the merged length stays ≤ the limit
   - exception: a single sentence longer than the limit but ≤ 2× the limit stays whole (one utterance)
   - a single sentence > 2× the limit is split at clause punctuation `、` `,` `;` `:` plus fullwidth `：` (a deliberate refinement of DESIGN §10.1's list, mirrored there), searching backward from the limit; splitting recurses on each remainder until every piece is ≤ the limit; a piece with no clause point is hard-split at the limit
   - `hardSplits` = the number of split boundaries created WITHOUT clause punctuation (0 when clause points sufficed)
   - `paragraphBreakAfter: true` on exactly the last utterance of each non-empty source paragraph (empty/whitespace paragraphs dropped first); reading order preserved; pure function (caller logs `{lang, hardSplits}` metadata only — never raw text).
3. `wav.ts` — contract constants exported and used everywhere (no magic numbers): `WAV_SAMPLE_RATE = 24000`, `WAV_CHANNELS = 1`, `WAV_BITS = 16`.
   - `writeSilenceWav(path, durationMs)`: canonical 44-byte RIFF/WAVE PCM header + `round(24000 * durationMs / 1000) * 2` zero bytes; header fields exact (ChunkSize, Subchunk2Size, byteRate 48000, blockAlign 2); deterministic bytes for a given duration.
   - `writePcm16Wav(path, samples: Int16Array, sampleRate: number)`: same canonical header shape with the given rate; used by providers converting engine output.
   - `wavDurationMs(path)`: parse RIFF — validate `RIFF`/`WAVE` magics; iterate chunks advancing `8 + size + (size % 2)` (odd-size padding) with bounds checks on every read; require a `fmt ` chunk with audioFormat PCM(1) and supported layout (mono/stereo, 8/16 bits — others → error) and a `data` chunk; `durationMs = round(dataBytes / (sampleRate * channels * bits/8) * 1000)`. Malformed/truncated/non-PCM → `EarmarkError AUDIO_BAD_WAV`.

## Acceptance Criteria

- [ ] Interface contract: tests include a compiling `FakeTtsProvider implements TtsProvider` writing silence WAVs (also exported from `test/util/` for issues 24/28) — proves the public types are implementable as intended.
- [ ] Splitter exact tables (both languages), including these fixed cases:
   - ja paragraph = five sentences of exactly 40 chars each → utterances of 120+80 chars (3 sentences merged, then 2)
   - ja single 130-char sentence → one utterance (≤ 2× rule), `hardSplits: 0`
   - ja single 250-char sentence with its only `、` at position 100 → pieces of exactly 100 / 120 / 30 chars (clause split at 100; clause-free 150-remainder hard-split at 120), `hardSplits: 1`
   - clause-free 300-char ja sentence → 120 / 120 / 60, `hardSplits: 2` (two boundaries created without clause punctuation)
   - en behaves with the 280 limit and space-joined merging
   - `paragraphBreakAfter` marks exactly one utterance per non-empty input paragraph; empty paragraphs produce none.
- [ ] Silence WAV: 500 ms file = canonical header + exactly 24,000 data bytes (byte-snapshot); `wavDurationMs` reads back 500.
- [ ] `writePcm16Wav` roundtrip: 1,000 samples at 24 kHz → duration ≈ 42 ms; byte-level header snapshot.
- [ ] Duration reader matrix: extra `LIST` chunk before `data` (fixture) → ok; odd-sized chunk followed by `data` → ok (padding honored); truncated file, wrong magic, missing `fmt `, missing `data`, audioFormat 3 (float) → `AUDIO_BAD_WAV` each.
- [ ] Roundtrip property: durations 1..2000 ms (seeded sample) write→read within ±1 ms.
- [ ] A real-ffmpeg-generated WAV fixture (`ffmpeg -f lavfi -i anullsrc=r=24000:cl=mono -t 0.5 -c:a pcm_s16le fixture.wav`, committed ~24 KB) parses correctly — interop proof.

## Validation

`vitest` as above. No manual steps.

## Dependencies

02 (config types only), 12 (`splitSentences` from `core/textseg.ts`), 14 (`ProviderHealth` from `core/provider.ts`).

## Non-goals

Any engine client (17/19/20); resampling (providers own conversion, via ffmpeg where needed — issues 19/20); AAC/m4a handling (issue 22).

## Design References

DESIGN §10.1, §9.3, §9.4; ADR-003 (WAV intermediate rationale), ADR-004.
