# VOICEVOX engine lifecycle (discover/autostart/stop)

## Summary

Implement `src/tts/voicevox-engine.ts`: detect whether the VOICEVOX engine is up, discover the engine binary across install variants, start it headless when `autoStart` is enabled, wait for readiness, and stop it cleanly if earmark started it.

## Context

The 06:00 launchd run cannot assume the user left the VOICEVOX GUI app open. DESIGN §10.3 specifies autostart; the binary location differs by install method (GUI app bundle vs bare engine) — a known unknown (ISSUE_PLAN U2) this issue must resolve empirically and document.

## Scope

- `src/tts/voicevox-engine.ts`, tests (fake engine child process + temp-fs discovery). No new runtime deps.

## Detailed Requirements

1. API — **provider-owned lifecycle** (ADR-004: the orchestrator only sees `TtsProvider`):
   ```ts
   ensureEngine(cfg: VoicevoxConfig, logger): Promise<EngineHandle>
   // EngineHandle = { startedByUs: boolean, stop(): Promise<void> }   // stop() is a no-op when startedByUs === false
   discoverEngineBinary(cfg, fsProbe?): Promise<{path: string} | {tried: string[]}>   // no-start discovery, exported for doctor (issue 27)
   ```
   This issue also wires `VoicevoxProvider` (issue 17): `prepare()` calls `ensureEngine` and stores the handle; `dispose()` calls `handle.stop()`. Issue 24 never touches engine lifecycle directly.
2. Health probe used everywhere: `GET <baseUrl>/version` (2 s) must return 2xx with a JSON **string** body (the VOICEVOX version shape). A 2xx with a different body shape (another service squatting the port, DESIGN §13.8) → treated as NOT a VOICEVOX engine: `TTS_UNAVAILABLE` with detail `unexpected service at <baseUrl> (not VOICEVOX)`, and autostart is NOT attempted (the port is occupied).
3. Flow of `ensureEngine`:
   - probe: engine up → return `{startedByUs:false}` (non-loopback baseUrl is fine here — reachable remote engine is allowed with the §13.6 warning from provider construction)
   - `autoStart === false` → throw `TTS_UNAVAILABLE`, remediation "start VOICEVOX or enable tts.ja.voicevox.autoStart"
   - autostart preconditions: baseUrl scheme `http:`, host loopback (`127.0.0.1`/`localhost`/`[::1]`), **explicit port present** — otherwise `TTS_UNAVAILABLE` "cannot autostart a non-local engine (<baseUrl>)" before any spawn
   - discover binary via `discoverEngineBinary` (first existing + executable wins):
     1. `cfg.enginePath` (if set; missing/non-executable → immediate error naming the path)
     2. `/Applications/VOICEVOX.app/Contents/Resources/vv-engine/run`
     3. `~/.local/opt/voicevox_engine/run`
     — **Implementation task**: verify path 2 against the current VOICEVOX.app bundle layout on the dev Mac; if it differs, update code AND this issue + DESIGN §10.3 in the same PR (U2 resolution). No match → `TTS_UNAVAILABLE` listing all tried paths.
   - spawn `execFile(binary, ['--host','127.0.0.1','--port', String(port)], …)` with stdout/stderr piped to debug log; keep the ChildProcess referenced (normal cleanup is `dispose()`/`finally`; if earmark itself is killed, a leftover engine process is tolerated — the next run's probe reuses it)
   - poll the health probe every 1 s up to 60 s; ready → `{startedByUs:true, stop}`; child early-exit or timeout → kill child, `TTS_UNAVAILABLE` with last stderr lines (≤ 500 chars, control-stripped).
4. `stop()`: SIGTERM → wait up to 10 s → SIGKILL fallback; always resolves; logs which signal sufficed.
5. Testability: binary discovery uses an injected `fs` accessor; spawn/poll tested against a **fake engine**: a committed tiny node script (`test/util/fake-voicevox-child.mjs`) that starts an HTTP `/version` server on the given `--port` after a configurable delay (env), enabling ready/slow/never-ready/early-exit/wrong-shape scenarios without VOICEVOX.

## Acceptance Criteria

- [ ] Engine-already-up path returns `startedByUs:false` and `stop()` does nothing.
- [ ] Wrong-shape `/version` response (HTML body via fake) → `TTS_UNAVAILABLE` `unexpected service…`, no spawn attempted, no synthesis reached.
- [ ] `autoStart:false` + engine down → `TTS_UNAVAILABLE` with the remediation string.
- [ ] Discovery order proven with temp-fs fakes (config path wins; app bundle second; bare install third; none → error listing all tried paths); `discoverEngineBinary` exported and callable without spawning (doctor contract).
- [ ] Fake-engine scenarios: ready-after-3s succeeds; never-ready times out at 60 s (accelerated via injected sleep) and child is killed; early-exit surfaces stderr excerpt; stop() SIGTERM path and SIGKILL fallback both covered.
- [ ] Autostart preconditions: non-loopback host, `https:` scheme, and missing port each → error before any spawn; reachable non-loopback engine (fake listening) is accepted without autostart.
- [ ] Provider wiring: `VoicevoxProvider.prepare()` triggers `ensureEngine` and `dispose()` stops a provider-started engine (integration with issue 17's provider using the fake engine).
- [ ] No shell involved anywhere (execFile with array args only).

## Validation

`vitest` with the fake child. Manual (dev Mac with VOICEVOX installed): `autoStart:true`, engine not running → scratch script runs `ensureEngine`, synthesizes one utterance via issue 17, `stop()`; verify no `vv-engine` process remains (`pgrep`); attach transcript + the resolved real bundle path (U2 evidence) to the PR.

## Dependencies

02, 04 (logger/EarmarkError), 17 (provider whose prepare/dispose this issue wires).

## Non-goals

Managing Ollama's lifecycle (user-managed service; doctor checks only); GUI app launching (`open -a VOICEVOX` — engine headless only); pidfile/daemon management; Windows/Linux paths.

## Design References

DESIGN §10.3, §12.1 (run-scoped TTS failures), §13.1 B6; ISSUE_PLAN U2; research/local-tts-selection.md.
