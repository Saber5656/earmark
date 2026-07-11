# `earmark run` orchestrator (lock, dry-run, idempotency)

## Summary

Implement `src/pipeline/orchestrate.ts`, `src/core/lock.ts`, and `src/cli/commands/run.ts`: the full morning pipeline of DESIGN §11.1 — lock, ingest, select, per-article content stages with failure isolation, script → TTS → assembly, transactional digest commit, run record with timings, and the `--dry-run/--force/--limit/--trigger` flags.

## Context

This wires every module built so far into the product's core loop (DESIGN §1.1 step 3). Correctness priorities: per-article isolation (one bad article never kills the digest), run-scoped vs article-scoped failure handling (§12.1), single-writer locking (§11.4), and date idempotency (§11.5) so launchd wake-delays and manual runs are safe.

## Scope

- `src/core/lock.ts`, `src/pipeline/orchestrate.ts`, `src/pipeline/notify.ts` (interface + console sink only), `src/cli/commands/run.ts`, tests with in-memory/fake providers.

## Detailed Requirements

1. Lock (`core/lock.ts`, DESIGN §11.4): `acquireRunLock(stateDir) → Release`:
   - create `<stateDir>/run.lock` with `O_EXCL` containing `{pid, startedAt}` JSON
   - on EEXIST: read it; stale iff `startedAt` older than 3 h OR pid not alive (`process.kill(pid, 0)` throws ESRCH) → unlink + retry once (log warn `stole stale lock`); else throw `EarmarkError LOCK_HELD` (exit code 4)
   - `Release()` unlinks; called in `finally`; unlink ENOENT tolerated.
2. Notify seam (`pipeline/notify.ts`): `interface NotifySink { notify(n: {title, body, kind: 'success'|'failure'}): Promise<void> }` + `ConsoleNotifySink` (logs info). osascript implementation arrives in issue 25; orchestrator depends only on the interface.
3. `runPipeline(deps, opts) → RunOutcome` — `deps` injects `{db, cfg, logger, providers: {translation, ttsFactory, engineLifecycle}, notify, now}` for testability; `opts = {dryRun, force, limit, trigger}`. Phases (§11.1) with per-phase timings collected into the §14.2 summary:
   1. acquire lock (skipped for dry-run? NO — dry-run also locks: it performs real ingest)
   2. `startRun(trigger)`
   3. ingest (issue 08) — report into summary
   4. idempotency check (issue 23): alreadyExists && !force → finish `no_articles`-like early success: status `success`, summary `{skipped: 'digest_exists'}`, log info, exit 0 (notification only at debug)
   5. select batch (limit)
   6. per article, sequentially: fetch → extract → speechify(placeholderLang=outputLanguage) → detect → needsTranslation ? translateDocument : passthrough → store via `updateContent` (title←extracted/translated, detected_language, content_text, translated_text, word_count) → collect `PreparedArticle`; stage throw → `classifyStageError`: article → `applyArticleFailure`, continue; run → abort to step 12 (fail path)
   7. prepared == 0 → finish status `no_articles` (summary lists failures), notify (only if failures occurred or `notifyOnEmpty`), exit 0
   8. dry-run → print plan (articles with title/lang/translated?/estimatedMinutes, planned chapter list, estimated total minutes = Σ estimates + fixed 1 min overhead), finish run status `success` with `summary.dryRun=true`, **no state changes beyond ingest + content caching in step 6** (article statuses untouched — content updates are allowed caching, statuses are not modified on dry-run: skip `applyArticleFailure`? NO — dry-run must not mutate retry counts: on stage error in dry-run, report in plan output but do NOT call applyArticleFailure. Implement via a `mutateOnFailure` flag.)
   9. script = `VerbatimScriptGenerator.generate({date, lang: outputLanguage, articles, stats})`
   10. TTS: provider = ttsFactory(outputLanguage) (voicevox via engineLifecycle `ensureEngine` when ja; kokoro/say per config when en); `checkAvailability` → fail-fast run-scoped; `prepare()`; split utterances per chapter (issue 16) preserving mapping {chapter, wav paths, paragraphBreakAfter}; synthesize sequentially into `<cacheDir>/work/<runId>/`
   11. assemble (issue 22) with `{date, sequence}` from step 4 → `AssembleResult`
   12. commit: `createDigestWithItems` (single transaction: digest row + items with chapter times from AssembleResult + markDigested each) — items positions: opening=0 is NOT stored; article chapters positions 1..N per DESIGN §5.1 comment (opening/closing not in digest_items)
   13. finish: status `success` (no article failures) / `partial` (≥1 failure) with full §14.2 summary; notify success template; run-scoped failure path: finish `failed`, articles stay queued (no retry increments for run-scoped aborts — verify none were applied), notify failure with remediation hint, exit 1
   14. `finally`: provider `dispose()` (engine stop), `pruneWorkDirs`, release lock.
4. `earmark run` command: flags per DESIGN §3; `--trigger` hidden default `cli`; maps `RunOutcome` to exit codes (0 success/partial/no_articles/skipped; 1 failed; 4 lock).
5. Crash-consistency: run row may remain `running` after SIGKILL — acceptable (§5.1 note); digest commit is atomic; no article can be `digested` without its digest row (single transaction).
6. Notifications content per DESIGN §14.3 exactly (count, minutes rounded, failed count).
7. All orchestrator logs structured `{runId, phase}`.

## Acceptance Criteria

- [ ] Happy path (fakes: passthrough translation, fake TTS writing silence wavs, real assembly stubbed with a fake returning planned times): 3 queued → digest row + 3 items + articles digested + run `success` + summary timings all present.
- [ ] Isolation: article 2 of 3 throws `FETCH_HTTP_404` → digest with 2 chapters, article 2 retry_count 1 still queued, run `partial`, closing stats fed correctly (verified via generator input capture).
- [ ] Run-scoped: TTS `checkAvailability` false → run `failed`, all articles untouched (retry counts unchanged), lock released, notify failure called.
- [ ] Idempotency: second run same day → early `skipped` success; `--force` → sequence 2 digest.
- [ ] Dry-run: no status/retry mutations (DB snapshot diff = only ingest inserts + content caching fields), plan printed, run recorded with `dryRun:true`.
- [ ] no_articles path (empty queue after ingest) → exit 0, `notifyOnEmpty:false` suppresses notification.
- [ ] Lock: concurrent second invocation exits 4; stale lock (dead pid fixture) stolen with warning.
- [ ] Crash simulation: throw between synthesis and commit → no digest rows, articles queued, work dir preserved.

## Validation

`vitest` orchestration suite with injected fakes (largest test file in the repo; table the scenarios). Manual: full real run on the dev Mac (real VOICEVOX + Ollama + 3 real articles incl. 1 English) — wave-4 gate; attach `earmark log --last --json` output to the PR.

## Dependencies

08, 10, 11, 12, 13, 14, 15, 16, 17, 18, 21, 22, 23 (19/20 exercised when `outputLanguage=en` configured).

## Non-goals

osascript notification implementation & `earmark log` (25); launchd installation (26); parallel article processing (v2 perf work); resuming a partially-synthesized run (work dir is disposable).

## Design References

DESIGN §11.1, §11.4, §11.5, §12, §14.2–14.3, §5.1–5.2; issues 16/21/22 models.
