# Kokoro TTS provider (en)

## Summary

Implement `src/tts/kokoro.ts`: the English `TtsProvider` running Kokoro-82M in-process via `kokoro-js` (ONNX), with model download/cache management under earmark's cache dir, contract-WAV output, and graceful failure guidance toward the `say` fallback. Resolves known unknown U1 (onnxruntime × current Node).

## Context

When `outputLanguage=en`, all narration and bodies are English; Kokoro gives far better long-form quality than macOS `say` (research/local-tts-selection.md). kokoro-js pulls `onnx-community/Kokoro-82M-v1.0-ONNX` on first use (~86–330 MB by dtype); that download must be explicit, logged, and cached deterministically.

## Scope

- `src/tts/kokoro.ts`, tests (unit with stubbed engine; opt-in real integration). Runtime dep added: `kokoro-js@^1` (exact-pin per DESIGN §13.7 since it drags onnxruntime prebuilds).

## Detailed Requirements

1. Constructor `new KokoroProvider(cfg.tts.en.kokoro, cacheDir, logger)`; `id='kokoro'`, `lang='en'`.
2. Model cache location: `<paths.cacheDir>/kokoro/` — configure kokoro-js/transformers-js to use it (the library resolves its cache via `@huggingface/transformers` env settings; **implementation task**: pin down the exact mechanism for the installed version — `env.cacheDir` or the `cache_dir` option — verify against the pinned version's docs/source, encode in code with a comment citing the version, and add a test asserting files land under our dir, not `~/.cache/huggingface`).
3. `prepare()`: loads the model (`KokoroTTS.from_pretrained(MODEL_ID, {dtype: cfg.dtype})`), triggering download when absent; log info before (`downloading Kokoro model (~<size est by dtype> MB, one-time)`) and after with elapsed; loaded instance retained on the provider.
4. `checkAvailability()`: does NOT download. Cache dir contains model files for the configured dtype → `{ok:true}`; absent → `{ok:false, detail:"Kokoro model not downloaded (~N MB on first run) — run: earmark doctor --fix-kokoro? NO"}` — detail text exactly: `"Kokoro model not cached; first run will download ~<N> MB"` with `ok:false` treated by doctor as WARN not FAIL (doctor issue 27 maps provider ids to severity).
5. `synthesize(utterance, outWavPath)`: `tts.generate(text, {voice: cfg.voice})` (API per pinned version) → returns audio (Float32 + sampling rate); convert Float32 [-1,1] → s16le with clamping; if sample rate ≠ 24000, resample is required — Kokoro outputs 24000 Hz natively; assert and hard-error `TTS_SYNTH_FAILED` if not (no resampler in-process; a mismatch means the library changed — fail loudly); write contract WAV via issue 16 header writer (extend `wav.ts` with `writePcm16Wav(path, int16Array, sampleRate)`); return duration from sample count. Sequential; utterance timeout 120 s via `Promise.race` → `TTS_SYNTH_FAILED` (no retry — in-process failures are deterministic).
6. Failure of module load (onnxruntime ABI mismatch with the running Node — U1): catch at `prepare()`/first import, throw `TTS_UNAVAILABLE` with remediation `"kokoro-js failed to load on this Node version — set tts.en.provider=say as a fallback (see docs/SETUP)"`; the error must surface identically through doctor.
7. Voice: config `voice` passed through (default `af_heart`); invalid voice → library error mapped to `TTS_SYNTH_FAILED` listing available voices if the API exposes them (best effort).
8. Unit-test seam: engine factory injected so tests stub `from_pretrained`/`generate` with a deterministic Float32 sine generator (duration = chars × 12 ms) — no download in CI.

## Acceptance Criteria

- [ ] Unit (stubbed): float→s16 conversion correctness (incl. clamp at ±1.0 and dithering-free rounding — snapshot 20 samples); WAV written at 24 kHz mono 16-bit; duration math from sample count exact; timeout path yields `TTS_SYNTH_FAILED`.
- [ ] `checkAvailability` cache-present/absent detection tested with fixture files in temp cacheDir.
- [ ] Load-failure path (stub throws on import/prepare) → `TTS_UNAVAILABLE` with the `say` remediation text.
- [ ] Real integration test exists, gated by `EARMARK_TEST_KOKORO=1` (downloads model, synthesizes "Good morning from earmark.", asserts duration > 500 ms and RMS > 0) — excluded from CI, run manually.
- [ ] Cache-dir redirection verified (no writes outside `<cacheDir>/kokoro/` in the stubbed download test; real test asserts the actual location).

## Validation

CI: stubbed unit suite. Manual (dev Mac): `EARMARK_TEST_KOKORO=1 npx vitest run -t kokoro-real`; listen to the produced WAV once; attach transcript, resolved cache path, and load-success-on-Node-<version> note (U1 evidence — if it FAILS on the dev Node version, stop and report: the owner decides whether `say` becomes the default; do not change defaults unilaterally).

## Dependencies

16.

## Non-goals

Japanese synthesis (Kokoro's ja support is not used in v1 — VOICEVOX owns ja); streaming synthesis; GPU/webgpu tuning; changing default en provider (owner decision if U1 fails).

## Design References

DESIGN §10.4, §9.3, §12.2; research/local-tts-selection.md; ISSUE_PLAN U1; ADR-001 (local model download allowance), ADR-004.
