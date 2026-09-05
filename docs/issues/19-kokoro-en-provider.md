# Kokoro TTS provider (en)

## Summary

Implement `src/tts/kokoro.ts`: the English `TtsProvider` running Kokoro-82M in-process via `kokoro-js` (ONNX), loaded lazily through an injectable engine factory, with model download/cache under earmark's cache dir, contract-WAV output (resampling via ffmpeg if ever needed), and failure guidance toward the `say` fallback. Resolves known unknown U1 (onnxruntime × current Node).

## Context

When `outputLanguage=en`, all narration and bodies are English; Kokoro gives far better long-form quality than macOS `say` (research/local-tts-selection.md). kokoro-js pulls `onnx-community/Kokoro-82M-v1.0-ONNX` on first use (~86–330 MB by dtype); that download must be explicit, logged, and land in earmark's own cache directory.

## Scope

- `src/tts/kokoro.ts`, tests (unit with stubbed engine; opt-in real integration). Runtime deps added: `kokoro-js@1.2.1` (exact pin per DESIGN §13.7 — it drags onnxruntime prebuilds) and `@huggingface/transformers` as a direct dependency at the same version range kokoro-js declares (needed to configure the cache location; see req 3).

## Detailed Requirements

1. Constructor:
   ```ts
   new KokoroProvider(cfg.tts.en.kokoro, deps: {
     cacheDir: string;                 // <paths.cacheDir>/kokoro
     ffmpegPath: string;               // for the (defensive) resample path
     logger: Logger;
     engineFactory?: () => Promise<KokoroEngine>;   // default: lazy dynamic import('kokoro-js') + from_pretrained
   })
   ```
   `id='kokoro'`, `lang='en'`. `kokoro-js` must NOT be imported at module top level — only inside the default engine factory (so import/ABI failures are catchable, req 6).
2. Default engine factory: `const { KokoroTTS } = await import('kokoro-js')`; `KokoroTTS.from_pretrained('onnx-community/Kokoro-82M-v1.0-ONNX', { dtype: cfg.dtype })`.
3. Cache redirection: before the dynamic import's first use, set `env.cacheDir = deps.cacheDir` via `import { env } from '@huggingface/transformers'` (kokoro-js resolves models through transformers.js, which honors `env.cacheDir`). Dependency-version note for implementers: keep the direct `@huggingface/transformers` range identical to the one in `kokoro-js@1.2.1`'s package.json so npm dedupes to a single instance (verify with `npm ls @huggingface/transformers`; a duplicated instance would leave the setting ineffective — treat as install-time failure).
4. `prepare()`: creates the engine via the factory (triggering download when absent); `logger.info` before (`downloading Kokoro model (~<size-by-dtype> MB, one-time)` — only when cache sentinel absent) and after with elapsed ms; on success writes sentinel file `<cacheDir>/.ready-<dtype>`; retains the engine instance.
5. `checkAvailability()` (no download, no import): sentinel `<cacheDir>/.ready-<dtype>` exists AND cache dir non-empty → `{ok:true, detail:"kokoro model cached (<dtype>)"}`; else `{ok:false, detail:"Kokoro model not cached; first run will download ~<N> MB"}` — doctor (issue 27) maps this provider's `ok:false` to WARN, not FAIL. (Size by dtype: fp32 ≈ 330, fp16 ≈ 170, q8 ≈ 90, q4 ≈ 50 — constants documented in code.)
6. Load failure (U1 — onnxruntime ABI mismatch with the running Node): any throw from the factory (import or `from_pretrained`) is caught in `prepare()` and rethrown as `EarmarkError TTS_UNAVAILABLE` with remediation `"kokoro-js failed to load on this Node version — set tts.en.provider=say as a fallback (see docs/SETUP.md)"`. Doctor does NOT attempt a load probe (issue 27 reports cache state only); this error surfaces at run time.
7. `synthesize(utterance, outWavPath)`: `engine.generate(text, {voice: cfg.voice})` (API per pinned version) → audio as Float32Array + sampling rate:
   - Float32 → Int16 conversion formula (exact): `s = Math.max(-32768, Math.min(32767, Math.round(f * 32767)))`
   - sample rate 24000 → write directly via `writePcm16Wav` (issue 16); any other rate → write a temp WAV at the native rate then resample with `execFile(ffmpegPath, ['-y','-hide_banner','-loglevel','error','-i', tmp, '-ar','24000','-ac','1','-c:a','pcm_s16le', outWavPath])` (DESIGN §9.3: providers deliver the contract format)
   - return duration from final sample count (or `wavDurationMs` after resample)
   - utterance timeout 120 s via `Promise.race` → `TTS_SYNTH_FAILED`; no retry (in-process failures are deterministic)
   - invalid voice → library error mapped to `TTS_SYNTH_FAILED`, listing available voices if the API exposes them (best effort).
8. Sequential synthesis only; `dispose()` releases the engine reference (no explicit teardown API assumed).

## Acceptance Criteria

- [ ] Unit (stubbed factory returning a deterministic sine generator, duration = chars × 12 ms): f32→s16 conversion snapshot incl. clamp at ±1.0 and the exact rounding formula (20-sample table); WAV written at 24 kHz mono 16-bit; duration math exact; timeout path → `TTS_SYNTH_FAILED`.
- [ ] Non-24 kHz stub output (e.g. 22050) triggers the ffmpeg resample argv (execFile spy asserts exact args) and yields a 24 kHz contract WAV.
- [ ] `prepare()` logging asserted with a stubbed logger: download-announce line only when sentinel absent; elapsed line always; sentinel written on success.
- [ ] `checkAvailability` matrix with fixture cache dirs: sentinel present+non-empty → ok; missing sentinel → the exact WARN detail string; never triggers the engine factory (spy).
- [ ] Factory throw in `prepare()` → `TTS_UNAVAILABLE` with the `say` remediation text; module import of `src/tts/kokoro.ts` itself never imports kokoro-js (verified via `--experimental-import-meta-resolve`-free static check or jest-style module spy — a simple test asserting the module loads with kokoro-js absent from a temp `node_modules` is acceptable).
- [ ] Real integration test gated by `EARMARK_TEST_KOKORO=1` (downloads model, synthesizes "Good morning from earmark.", asserts duration > 500 ms and non-zero RMS, asserts model files under `deps.cacheDir` and NOT under `~/.cache/huggingface`) — excluded from CI.

## Validation

CI: stubbed unit suite. Manual (dev Mac): `EARMARK_TEST_KOKORO=1 npx vitest run -t kokoro-real`; listen to the produced WAV once; attach transcript, resolved cache path, and a load-success-on-Node-`$(node --version)` note (U1 evidence). If loading FAILS on the dev Node version: stop, report to the owner with the error (the owner decides whether `say` becomes the en default) — do not change defaults unilaterally.

## Dependencies

16 (`writePcm16Wav`, wav contract, provider interface), 02 (config), 04 (logger).

## Non-goals

Japanese via Kokoro (VOICEVOX owns ja); streaming synthesis; GPU/webgpu tuning; doctor-side load probing (cache check only, issue 27); changing the default en provider.

## Design References

DESIGN §10.4, §9.3, §12.1–12.2; research/local-tts-selection.md; ISSUE_PLAN U1; ADR-001 (local model download allowance), ADR-004.
