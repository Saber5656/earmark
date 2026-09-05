# Research: Local TTS engine selection (ja / en)

- Date: 2026-07-11
- Method: prior engine knowledge + npm registry verification on 2026-07-11 (`npm view <pkg> version`); runtime quality to be re-validated on real hardware during issues 17/19 (listening check).
- Feeds: DESIGN §10, ADR-004.

## Requirements

Local-only (ADR-001), free, runs on Apple Silicon macOS, callable from Node.js, acceptable long-form quality: Japanese (primary) and English (when `outputLanguage=en`).

## Japanese candidates

| Engine | Integration | Quality (ja) | Notes |
|---|---|---|---|
| **VOICEVOX** (chosen) | Local REST engine `127.0.0.1:50021` (`/audio_query` → `/synthesis`, WAV out) | High, natural; many voices (style ids) | Free incl. commercial use w/ credit terms per voice library; GUI app bundles the engine binary; engine can run headless. De-facto standard for local ja TTS. |
| Style-Bert-VITS2 | Python server, manual model setup | Very high | Heavy setup, model licensing varies — rejected for default. |
| macOS `say` (Kyoko) | builtin CLI | Poor for long-form ja | Rejected as ja default. |
| OpenJTalk | CLI | Dated, robotic | Rejected. |

Default voice: speaker/style id `3` (Zundamon, normal) — id verified against engine `/speakers` at implementation time; `speedScale` exposed in config.

## English candidates

| Engine | Integration | Quality (en) | Notes |
|---|---|---|---|
| **Kokoro-82M** (chosen) | `kokoro-js@1.2.1` (npm, verified) runs ONNX in-process via onnxruntime; model `onnx-community/Kokoro-82M-v1.0-ONNX`, ~86–330 MB by dtype | High for 82M; well-reviewed long-form | Apache-2.0. First-use download → cache under earmark's cache dir; doctor pre-warns. Node-version compatibility of onnxruntime-node is a known unknown (ISSUE_PLAN). |
| **macOS `say`** (chosen as fallback) | builtin `/usr/bin/say` → AIFF → ffmpeg to WAV | Mediocre but serviceable | Zero-install; keeps en path working if kokoro-js breaks. Config `tts.en.provider="say"`. |
| Piper | external binary + voice files | Good, fast | Extra binary distribution burden vs kokoro-js in-process — not default; possible future provider. |

## Decision summary

- ja: VOICEVOX provider + engine lifecycle management (autostart when installed as GUI app or bare engine).
- en: kokoro-js default, `say` fallback provider selectable in config.
- Both produce the shared WAV contract (24 kHz/16-bit/mono, DESIGN §9.3); `say` output is resampled via ffmpeg.

## Risks / follow-ups

- kokoro-js on Node 26 (onnxruntime-node ABI) — verify early in issue 19; fallback path exists.
- VOICEVOX GUI-app engine path differs across install methods — discovery order specified in DESIGN §10.3; doctor reports what it found.
- Long-input stability: both engines are called per-utterance (≤ 120/280 chars), avoiding known long-input degradation.
