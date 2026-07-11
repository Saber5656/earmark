# E2E pipeline test with mock engines

## Summary

Build the CI-safe end-to-end suite: fixture articles served over loopback HTTP, mock VOICEVOX + mock Ollama engines, real `earmark run` invocations of the **built CLI as a child process** in an isolated sandbox, and ffprobe-based assertions on the produced `.m4a` plus DB state assertions — covering both `ja` (mock VOICEVOX) and `en` (real macOS `say`) output languages.

## Context

Unit and module tests prove parts; this proves the wiring (DESIGN §16 E2E row) and becomes the regression net for every later change. It must run on `macos-14` CI with zero external network and no VOICEVOX/Ollama/kokoro installs. The `en` scenario uses the `say` provider (present on CI runners), closing the v1 promise that both output languages work end-to-end.

## Scope

- `test/e2e/run.e2e.test.ts`, `test/util/sandbox.ts` (env builder), `test/util/net-guard.mjs` (child-process network guard), fixture site server `test/util/fixture-site.ts`; reuses `mock-voicevox.ts` (issue 17) and `mock-ollama.ts` (issue 15). CI workflow update if ffmpeg is missing on the runner image.

## Detailed Requirements

1. CLI invocation (fixed): `npm run build` once per suite, then spawn `node dist/cli/index.js <args>` as a child process — exit codes and stdio are part of the contract under test. No `tsx` path.
2. Sandbox (`test/util/sandbox.ts`): temp dirs for `EARMARK_CONFIG_DIR`, data/state/cache/output/inbox; writes a config pointing all `paths.*` at temp dirs, `network.allowPrivateNetworks: true` (test-only deviation — loopback fixtures), provider base URLs at the mocks' ephemeral ports, `tts.ja.voicevox.autoStart: false`, `notifications.enabled: false`, per-scenario `outputLanguage`/`digest.maxArticles`. Returns helpers `runCli(args) → {code, stdout, stderr}` and direct better-sqlite3 read access to the sandbox DB.
3. Child-process network guard: every `runCli` sets `NODE_OPTIONS=--import <abs path>/test/util/net-guard.mjs`; the guard module (test-only, never shipped — lives under `test/`, excluded from the npm package by `files`) installs an undici global dispatcher/interceptor that throws on any request whose host is not `127.0.0.1`/`::1` and logs the offender. Product code gets no test hooks (guard is pure preload).
4. Fixture site (`fixture-site.ts`, loopback `node:http`): `/ja-article-1`, `/ja-article-2`, `/en-article`, `/redirect` (302 → `/ja-article-1`), `/paywall` (< 200 chars extractable), `/huge` (streams 20 MB), correct content-types.
5. Scenario A — primary ja digest (`outputLanguage: ja`, maxArticles 10):
   - seed: `runCli(['add', <ja1>])`, `add <en-article>`, `add <paywall>`; one inbox JSON file (ja2) in the sandbox inbox
   - `runCli(['run'])` → exit 0
   - DB: ja1/ja2/en `digested`, paywall `queued` retry_count 1; digest row + **3 digest_items** (article chapters only); run `partial`, summary counts `{ingested: 1, selected: 4, prepared: 3}`
   - m4a at `<output>/earmark-<today>.m4a`; ffprobe: **5 chapters** — `オープニング`, 3 article titles (the en article's chapter title carries the mock-translation `⟪ja⟫` prefix, proving the translation path), `クロージング`; chapter times strictly monotonic and non-overlapping; tags `album=earmark`, `date=<today>`
   - chapter-time cross-check (determinism, two layers): (a) article-chapter START/END from ffprobe match the DB `digest_items.start_ms/end_ms` within ±200 ms; (b) total duration within ±5% of an in-test expectation computed as follows — read `articles.translated_text`/`content_text` and titles from the sandbox DB after the run, rebuild the full `DigestScript` by importing `VerbatimScriptGenerator` + i18n (same date/lang/stats), split every segment with the imported utterance splitter, expected synth time = Σ(utterance text length × 10 ms) (the mock VOICEVOX duration formula, applied to the exact utterance texts) + configured paragraph/chapter gaps for the whole plan including opening/closing chapters
   - idempotency: second `runCli(['run'])` → exit 0, no new digest; then seed one fresh article (`add` a new fixture URL) and `runCli(['run','--force'])` → `-2` file with 3 chapters (opening + 1 article + closing). (Without the fresh seed only the paywall article remains queued and a forced run would end `no_articles`.)
6. Scenario B — run-scoped TTS failure (fresh sandbox): seed only `/ja-article-1`; mock VOICEVOX `mode=500` → `runCli(['run'])` exit 1; run row `failed`; article still `queued` with `retry_count` **0** (run-scoped aborts stage nothing — issue 24); no file in the output dir (no `.part` leftovers).
7. Scenario C — redirect + size cap (fresh sandbox): seed `/redirect` and `/huge` → run → **exit 0, run status `partial`**; `/redirect` article `digested` (its `articles.url` remains the original `/redirect` URL; final-URL persistence is intentionally NOT asserted — no such column), `/huge` `queued` retry_count 1 with `last_error` starting `FETCH_TOO_LARGE`.
8. Scenario D — en digest via `say` (fresh sandbox; darwin-only, runs on CI): `outputLanguage: 'en'`, `tts.en.provider: 'say'`; seed `/ja-article-1` (gets mock-translated to `⟪en⟫…`) and `/en-article` → run → exit 0; ffprobe: 4 chapters (`Opening`, 2 articles, `Closing`); tags `title` prefix `earmark`; audio duration > 5 s (real speech). No strict duration math (say timing is not deterministic).
9. Runtime budget: full e2e file < 180 s on CI. CI workflow: verify ffmpeg presence on the runner image at implementation time; add a `brew install ffmpeg` step if absent (document the finding in a workflow comment).

## Acceptance Criteria

- [ ] Scenarios A–D pass locally and on `macos-14` CI, twice consecutively (flake check — link both runs in the PR).
- [ ] Network guard proven: a deliberate test targeting `http://192.0.2.1/` (TEST-NET-1, non-routable even if the guard were broken) through the guarded child fails with the guard's error before any connection attempt (meta-test).
- [ ] ffprobe assertions as specified per scenario (chapter counts incl. opening/closing, localized titles, `⟪ja⟫`/`⟪en⟫` markers, monotonic times, tags, DB↔ffprobe chapter-time cross-check in A).
- [ ] DB assertions via direct sandbox-DB reads (not CLI output).
- [ ] Exit codes asserted for every invocation.
- [ ] Sandbox writes nothing outside its temp roots (`~/Library/LaunchAgents` and the real `~/.config/earmark` byte-identical before/after suite).

## Validation

CI green ×2 links in PR; local `npm test` includes the suite.

## Dependencies

24 (pipeline), 15/17 (mocks), 20 (`say` provider for scenario D).

## Non-goals

Real VOICEVOX/Ollama/kokoro tests (manual wave gates own those); performance benchmarking (DESIGN §17 uses real runs); launchd-triggered e2e (issue 26 manual validation).

## Design References

DESIGN §16 (E2E row, no-network rule), §9.1–9.2 (chapter structure incl. opening/closing), §9.4, §11.5, §12; issues 15/17 mock contracts, issue 24 `RunOutcome`.
