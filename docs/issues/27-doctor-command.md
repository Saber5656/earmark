# `earmark doctor` environment diagnosis

## Summary

Implement `src/doctor/checks.ts` and `src/cli/commands/doctor.ts`: run the full DESIGN §15 check list with PASS/WARN/FAIL results and one-line remediations, aggregate provider `checkAvailability` calls, and exit 3 when any check FAILs.

## Context

VOICEVOX and Ollama are not preinstalled on user machines (nor on the dev Mac today); doctor is the setup companion (README's first instruction after install) and the first debugging step for 06:00 failures. Every provider already exposes `checkAvailability` (ADR-004); doctor is the aggregator plus environment checks.

## Scope

- `src/doctor/checks.ts` (each check a `DoctorCheck` object), `src/cli/commands/doctor.ts`, tests with injected fakes.

## Detailed Requirements

1. Model: `DoctorCheck = { id, title, run(ctx) → {status: 'pass'|'warn'|'fail', detail, remediation?} }`; ctx carries cfg, logger, exec/http/fs seams. Checks run sequentially, each wrapped so a thrown error becomes `fail` with the message (doctor never crashes).
2. Check list (ids fixed; conditional checks noted):
   | id | severity on problem | logic |
   |---|---|---|
   | `node-version` | fail | `process.versions.node` major ≥ 22 |
   | `config-valid` | fail | config loads (already true if we got here — re-validate file explicitly) |
   | `dirs-writable` | fail | data/state/cache dirs creatable+writable (touch probe) |
   | `ffmpeg` | fail | `execFile(ffmpegPath, ['-version'])` ok; detail = first line; same for `ffprobe` (one combined check, two probes) |
   | `output-dir` | fail | resolvedOutputDir creatable+writable |
   | `inbox-dir` | warn | resolvedInboxDir exists (missing → remediation: run `earmark ingest` once / check iCloud Drive; custom non-iCloud path missing → same) |
   | `db` | fail | openDb succeeds; schema version current; detail row counts |
   | `tts-ja` | fail (only when `outputLanguage==='ja'`) | VOICEVOX `checkAvailability`; if unreachable AND `autoStart` → engine discovery (issue 18 exported `discoverEngineBinary`) result: found → pass with detail `engine down, autostart ready: <path>`; neither → fail with install remediation |
   | `tts-ja-speaker` | warn (when tts-ja passed via live engine) | configured speaker id present in `/speakers` (issue 17 `listSpeakers`); mismatch → warn listing 3 nearest ids (U8) |
   | `tts-en` | fail when `outputLanguage==='en'`, warn otherwise-skip | kokoro: cache-present check (absent → **warn** with size note per issue 19); say: availability check; load-failure (U1) → fail with `say` fallback remediation |
   | `translation` | warn | Ollama `checkAvailability` (unreachable/model-missing → warn: translation only needed for cross-language articles; remediation strings from issue 15 verbatim) |
   | `schedule` | warn | issue 26 status: not installed → warn `run earmark schedule install`; drift → warn with details; installed+clean → pass |
   | `disk-space` | warn | ≥ 1 GB free on cacheDir volume (`statfs` via `fs.statfs`) |
   | `lock` | warn | stale lock file present → warn `stale lock from pid <p> — will be reclaimed next run` |
3. Ordering: environment (node→config→dirs→ffmpeg→db) then providers then schedule/disk/lock — remediation earlier items first.
4. Output (human): aligned `✓/⚠/✗ <title>: <detail>` lines, remediation indented on problem lines; summary footer `X passed, Y warnings, Z failed`; `--json`: array of results + summary. Exit code: any fail → 3; else 0 (warnings don't fail — they'd block first-run UX).
5. Conditional logic: checks not applicable to the active config (e.g. `tts-en` when ja-only) run as `pass` with detail `skipped (not configured)` — visible, not hidden (users toggling `outputLanguage` see what would be needed? NO — skipped checks that would apply to the OTHER language render as informational `skipped`; keep output honest and stable).
6. Every remediation string matches the corresponding provider/setup doc wording (issues 15/17/18/19/20/26; SETUP doc issue 30 will quote doctor output — keep strings stable).

## Acceptance Criteria

- [ ] Fake-injected matrix: all-green env → exit 0 with 0 warnings; each check individually forced to its problem state produces the specified status/remediation and correct exit code.
- [ ] Conditional matrix: `outputLanguage=ja` → tts-en shows skipped-pass, tts-ja active; `en` → inverse; translation always evaluated as warn-severity.
- [ ] kokoro-cache-missing is WARN not FAIL; VOICEVOX down with discoverable engine is PASS (autostart-ready detail).
- [ ] Human output snapshot (fixed fakes) and `--json` schema stable.
- [ ] A check that throws (injected) renders fail without aborting subsequent checks.
- [ ] Real-machine smoke on CI (macos-14: node/ffmpeg/dirs/db pass; voicevox/ollama absent → their statuses appear without crash; overall exit 3 is acceptable in this smoke — assert output shape, not exit).

## Validation

`vitest` matrix + CI smoke. Manual: run on the dev Mac before and after installing VOICEVOX/Ollama during wave-4; attach both outputs to the PR (they double as SETUP doc screenshots source, issue 30).

## Dependencies

15, 18, 19, 20, 26 (checkAvailability/status providers), plus core issues transitively.

## Non-goals

Auto-fixing (`--fix`) anything; installing software; network reachability tests beyond the two loopback services; doctor-driven config migration.

## Design References

DESIGN §15, §12.4 (exit 3), §13.6; ADR-004; ISSUE_PLAN U1/U2/U8.
