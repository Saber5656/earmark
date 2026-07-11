# VOICEVOX TTS provider (ja)

## Summary

Implement `src/tts/voicevox.ts`: the Japanese `TtsProvider` calling a local VOICEVOX engine's REST API (`/audio_query` → `/synthesis`), applying the configured speaker/style id and `speedScale`, writing contract WAVs, with timeout/retry and a mock-engine test server.

## Context

VOICEVOX is the chosen ja engine (research/local-tts-selection.md). The engine runs at `http://127.0.0.1:50021` (config `tts.ja.voicevox`); process lifecycle is issue 18 — this issue assumes a reachable engine and stays a pure HTTP client behind the issue-16 interface.

## Scope

- `src/tts/voicevox.ts`, `test/util/mock-voicevox.ts`, tests.

## Detailed Requirements

1. Constructor `new VoicevoxProvider(cfg.tts.ja.voicevox, logger)`; `id='voicevox'`, `lang='ja'`; non-loopback baseUrl → §13.6 warning once.
2. `checkAvailability()`: `GET <baseUrl>/version` (2 s timeout) → 200 with a JSON string body → `{ok:true, detail:"voicevox <version>"}`; else `{ok:false, detail:"VOICEVOX engine not reachable at <baseUrl> — see docs/SETUP (or set tts.ja.voicevox.autoStart)"}`.
3. `synthesize(utterance, outWavPath)` per DESIGN §10.2:
   - `POST <baseUrl>/audio_query?speaker=<id>&text=<urlencoded utterance.text>` (empty body) → JSON audio query
   - mutate: `speedScale = cfg.speedScale`; leave all other fields untouched
   - `POST <baseUrl>/synthesis?speaker=<id>` with `content-type: application/json` body = the mutated query, `accept: audio/wav` → binary WAV → write to `outWavPath` (temp name + rename for atomicity within the work dir)
   - verify/normalize format: parse header via issue 16 `wavDurationMs`/format check; engine output is 24 kHz mono 16-bit by default — if sampleRate ≠ 24000 (engine configured differently), set `outputSamplingRate=24000` in the audio query **preemptively** (always set it, plus `outputStereo=false`) so the contract holds by construction
   - return `{durationMs}` from the written file
   - per-utterance timeout 60 s on each HTTP call; on 5xx/network error/timeout: retry the whole utterance once; second failure → `EarmarkError TTS_SYNTH_FAILED` with utterance index (run-scoped handling per DESIGN §12.1 happens in the orchestrator)
   - 4xx (e.g. invalid speaker) → no retry, `TTS_SYNTH_FAILED` with the engine's error body (≤ 200 chars) in the message
   - calls strictly sequential (no concurrency), matching DESIGN §10.2.
4. Speaker id: used verbatim from config; a helper `listSpeakers()` (`GET /speakers` → `[{name, styles:[{id,name}]}]` simplified) is exported for doctor (issue 27) to display/validate (U8).
5. Mock engine (`test/util/mock-voicevox.ts`): `/version` returns `"mock-0.0.0"`; `/audio_query` returns a minimal valid query JSON echoing text length; `/synthesis` returns a generated silence WAV whose duration = `text.length * 10` ms (deterministic; uses issue 16 generator); failure modes: `mode=500-once` (first synthesis 500, then ok), `mode=500`, `mode=slow`, `mode=bad-speaker` (400).
6. Logs: per-utterance debug `{index, chars, elapsedMs}`; never log utterance text above debug.

## Acceptance Criteria

- [ ] Happy path against mock: N utterances → N WAV files at the contract format; durations equal `chars*10ms` ±1 ms; `audio_query` request had `outputSamplingRate` forced to 24000 and `outputStereo=false` (mock asserts).
- [ ] `speedScale` from config appears in the synthesis request body (mock captures and asserts).
- [ ] `500-once` succeeds via retry (mock counts 2 synthesis calls); persistent `500` → `TTS_SYNTH_FAILED` after exactly 2 attempts; `bad-speaker` → fails without retry.
- [ ] `checkAvailability` ok/unreachable both produce the specified details.
- [ ] URL-encoding proven with text containing `&`, `%`, spaces, newlines, and emoji (mock echoes received text; roundtrip equality).
- [ ] Atomic write: no partial `.wav` left when synthesis is interrupted mid-write (inject write failure).

## Validation

`vitest` vs mock engine. Manual (dev Mac, VOICEVOX installed): scratch script synthesizes 「こんにちは、イヤーマークのテストです。」 to a WAV; listen once; attach `afinfo` output to PR. (Real-engine full-digest listening happens at wave-4 gate.)

## Dependencies

16.

## Non-goals

Engine start/stop/discovery (18); accent/pitch tuning beyond `speedScale`; speaker enumeration UX (doctor, 27); multi-engine ja support.

## Design References

DESIGN §10.2, §9.3, §12.1–12.2, §13.6; research/local-tts-selection.md; ISSUE_PLAN U8.
