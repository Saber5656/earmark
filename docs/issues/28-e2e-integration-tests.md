# E2E pipeline test with mock engines

## Summary

Build the CI-safe end-to-end test: fixture articles served over loopback HTTP, mock VOICEVOX + mock Ollama engines, a real `earmark run` invocation through the CLI in an isolated HOME/config sandbox, and ffprobe-based assertions on the produced `.m4a` plus DB state assertions.

## Context

Unit and module tests prove parts; this proves the wiring (DESIGN §16 E2E row) and becomes the regression net for every later change. It must run on `macos-14` CI with zero external network and no VOICEVOX/Ollama/kokoro installs (mocks + `say`-free path: ja pipeline with mock VOICEVOX).

## Scope

- `test/e2e/run.e2e.test.ts`, `test/util/sandbox.ts` (env builder), fixture site server `test/util/fixture-site.ts`; reuses `mock-voicevox.ts` (issue 17) and `mock-ollama.ts` (issue 15). CI workflow update if needed (ffmpeg presence).

## Detailed Requirements

1. Sandbox (`test/util/sandbox.ts`): temp dirs for `EARMARK_CONFIG_DIR`, data/state/cache/output/inbox (config written pointing all `paths.*` at temp, `network.allowPrivateNetworks: true` — loopback fixtures, note this deviation is test-only, `translation.ollama.baseUrl`/`tts.ja.voicevox.baseUrl` at the mocks' ephemeral ports, `autoStart:false`, `notifications.enabled:false`, `digest.maxArticles` as per scenario); helper runs the CLI **as a child process** (`node dist/cli/index.js …` after `npm run build`, or via `tsx src/cli/index.ts` — pick one and pin; child-process execution is required: exit codes and stdio contracts are part of the test).
2. Fixture site: loopback HTTP server serving `/ja-article-1`, `/ja-article-2` (ja fixtures), `/en-article` (en fixture), `/redirect` (302 → ja-article-1), `/paywall` (< 200 chars), `/huge` (streams 20 MB), with correct content-types.
3. Scenario A (primary, `outputLanguage: ja`, maxArticles 10):
   - seed: `earmark add` ×3 (ja1, en, paywall) + 1 inbox JSON file (ja2) in temp inbox
   - run `earmark run --trigger cli`
   - assert exit 0; DB: ja1/ja2/en `digested`, paywall `queued` retry_count 1; digest row + 3 items with monotonic times; run row `partial` with summary counts `{ingested:1, selected:4, prepared:3}`
   - m4a: exists at `<output>/earmark-<today>.m4a`; ffprobe chapters = 3 titles matching (en article's chapter title carries the mock-translated marker `⟪ja⟫…` proving the translation path ran); duration equals mock-deterministic expectation ±5% (mock synth 10 ms/char + configured gaps — compute expected from the plan)
   - second run same day → exit 0, no second digest (idempotency); `--force` → `-2` file.
4. Scenario B (failure isolation): mock VOICEVOX in `mode=500` → run exits 1, run row `failed`, all articles still `queued` with retry counts unchanged from scenario A? (fresh sandbox: retry_count 0), no partial files in output dir.
5. Scenario C (redirect + size cap): add `/redirect` (succeeds, final URL stored? — assert digested) and `/huge` (fails `FETCH_TOO_LARGE`, retry_count 1).
6. Determinism: mock synth duration formula + fixed gap config → assert exact expected chapter STARTs (±200 ms tolerance from AAC priming/encoder delay only).
7. Runtime budget: full e2e file < 120 s on CI (silence audio is small; loudnorm on ~1 min audio is fast).
8. CI: ensure workflow installs ffmpeg (`brew install ffmpeg` step if the runner image lacks it — check at implementation; document the finding in the workflow file comment).

## Acceptance Criteria

- [ ] Scenarios A/B/C pass locally and on `macos-14` CI, no external network (fixture/mocks bound to 127.0.0.1; a network-guard dispatcher asserts no other hosts contacted).
- [ ] ffprobe assertions cover: chapter count, titles (incl. translated marker), monotonic non-overlapping times, tags (`album=earmark`, `date`), duration tolerance.
- [ ] DB assertions via direct better-sqlite3 read of the sandbox DB (not via CLI) for states/retry counts.
- [ ] Exit-code contract asserted for every scenario (0/0-idempotent/1).
- [ ] Sandbox leaves no files outside its temp roots (assert `~/Library/LaunchAgents` untouched, real HOME config untouched).

## Validation

CI run green ×2 consecutive (flake check) — link both runs in the PR. Local `npm test` including e2e.

## Dependencies

24 (whole pipeline callable), 15/17 (mocks exist).

## Non-goals

Real-engine tests (manual wave gates own those); en-output e2e via kokoro (U1-dependent; `say`-based en e2e may be added here if trivially stable on CI, else deferred to a follow-up issue); performance benchmarking (DESIGN §17 uses real runs).

## Design References

DESIGN §16 (E2E row, no-network rule), §9.4, §11.5, §12; issues 15/17 mock contracts.
