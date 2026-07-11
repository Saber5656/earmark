# VOICEVOX engine lifecycle (discover/autostart/stop)

## Summary

Implement `src/tts/voicevox-engine.ts`: detect whether the VOICEVOX engine is up, discover the engine binary across install variants, start it headless when `autoStart` is enabled, wait for readiness, and stop it cleanly if earmark started it.

## Context

The 06:00 launchd run cannot assume the user left the VOICEVOX GUI app open. DESIGN §10.3 specifies autostart; the binary location differs by install method (GUI app bundle vs bare engine) — a known unknown (ISSUE_PLAN U2) this issue must resolve empirically and document.

## Scope

- `src/tts/voicevox-engine.ts`, tests (fake engine child process + temp-fs discovery). No new runtime deps.

## Detailed Requirements

1. API:
   ```ts
   ensureEngine(cfg: VoicevoxConfig, logger): Promise<EngineHandle>
   // EngineHandle = { startedByUs: boolean, stop(): Promise<void> }
   ```
   Used by the orchestrator around the ja-TTS phase; `stop()` is a no-op when `startedByUs === false`.
2. Flow of `ensureEngine`:
   - probe `GET <baseUrl>/version` (2 s): up → return `{startedByUs:false}`
   - `autoStart === false` → throw `EarmarkError TTS_UNAVAILABLE` with remediation "start VOICEVOX or enable tts.ja.voicevox.autoStart"
   - discover binary (first existing + executable wins):
     1. `cfg.enginePath` (if set; missing/non-executable → immediate error naming the path)
     2. `/Applications/VOICEVOX.app/Contents/Resources/vv-engine/run`
     3. `~/.local/opt/voicevox_engine/run`
     — **Implementation task**: verify path 2 against the current VOICEVOX.app bundle layout on the dev Mac at implementation time; if it differs, update this list in code AND in this issue file + DESIGN §10.3 in the same PR (U2 resolution), keeping config override as the escape hatch.
   - spawn via `execFile(binary, ['--host','127.0.0.1','--port', String(port-from-baseUrl)], {stdio-to-debug-log})`; do not detach; keep the ChildProcess
   - poll `/version` every 1 s up to 60 s; ready → return `{startedByUs:true, stop}`; child exits early or 60 s elapse → kill child, throw `TTS_UNAVAILABLE` including last stderr lines (≤ 500 chars, control-stripped)
3. `stop()`: SIGTERM → wait up to 10 s for exit → SIGKILL fallback; always resolves; logs which signal sufficed.
4. Port parsing: derived from `baseUrl` (default 50021); baseUrl with non-loopback host + autoStart → error (cannot start a remote engine) with clear message.
5. Crash-safety: if earmark is killed mid-run, the child engine may orphan — mitigate by `child.unref()` NOT being called (keeps lifetime coupled) and documenting that a leftover engine process is harmless (idempotent probe reuses it next run). No pidfile in v1.
6. Testability: binary discovery uses an injected `fs` accessor; spawn/poll tested against a **fake engine**: a committed tiny node script (`test/util/fake-voicevox-child.mjs`) that starts an HTTP `/version` server on the given `--port` after a configurable delay (env), enabling ready/slow/never-ready/early-exit scenarios without VOICEVOX.

## Acceptance Criteria

- [ ] Engine-already-up path returns `startedByUs:false` and `stop()` does nothing.
- [ ] `autoStart:false` + engine down → `TTS_UNAVAILABLE` with the remediation string.
- [ ] Discovery order proven with temp-fs fakes (config path wins; app bundle second; bare install third; none → error listing all tried paths).
- [ ] Fake-engine scenarios: ready-after-3s succeeds; never-ready times out at 60 s (test clock accelerated via injected sleep) and child is killed; early-exit surfaces stderr excerpt; stop() SIGTERM path and SIGKILL fallback both covered.
- [ ] Non-loopback baseUrl + autoStart → error before any spawn.
- [ ] No shell involved anywhere (execFile with array args only).

## Validation

`vitest` with the fake child. Manual (dev Mac with VOICEVOX installed): `autoStart:true`, engine not running → scratch script runs `ensureEngine`, synthesizes one utterance via issue 17, `stop()`; verify no `vv-engine` process remains (`pgrep`); attach transcript + the resolved real bundle path (U2 evidence) to the PR.

## Dependencies

02, 17.

## Non-goals

Managing Ollama's lifecycle (user-managed service; doctor checks only); GUI app launching (`open -a VOICEVOX` — engine headless only); pidfile/daemon management; Windows/Linux paths.

## Design References

DESIGN §10.3, §12.1 (run-scoped TTS failures), §13.1 B6; ISSUE_PLAN U2; research/local-tts-selection.md.
