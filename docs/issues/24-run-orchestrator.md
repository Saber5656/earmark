# `earmark run` orchestrator (lock, dry-run, idempotency)

## Summary

Implement `src/pipeline/orchestrate.ts`, `src/core/lock.ts`, and `src/cli/commands/run.ts`: the full morning pipeline of DESIGN §11.1 — lock, ingest, select, per-article content stages with failure isolation (staged, terminal-commit persistence), script → TTS → assembly, transactional digest commit, run record with timings, and the `--dry-run/--force/--limit/--trigger` flags.

## Context

This wires every module built so far into the product's core loop (DESIGN §1.1 step 3). Correctness priorities: per-article isolation (one bad article never kills the digest), run-scoped vs article-scoped failure handling (§12.1) with **no retry-count mutations on run-scoped aborts**, single-writer locking (§11.4), and date idempotency (§11.5) so launchd wake-delays and manual runs are safe.

## Scope

- `src/core/lock.ts`, `src/pipeline/orchestrate.ts`, `src/pipeline/notify.ts` (interface + console sink only), `src/cli/commands/run.ts`, tests with in-memory/fake providers.

## Detailed Requirements

1. Lock (`core/lock.ts`, DESIGN §11.4): `acquireRunLock(stateDir) → Release`:
   - create `<stateDir>/run.lock` with `O_EXCL` containing `{pid, startedAt}` JSON
   - on EEXIST: read it; stale iff `startedAt` older than 3 h OR pid dead (**only** `process.kill(pid, 0)` throwing `ESRCH` means dead; `EPERM` means alive) OR the lock file is unreadable/malformed JSON (treated as stale with a warning) → unlink + retry once (warn `stole stale lock`); else throw `EarmarkError LOCK_HELD` (exit code 4)
   - `Release()` unlinks (ENOENT tolerated); called in `finally`.
2. Notify seam (`pipeline/notify.ts`): `interface NotifySink { notify(n: {title, body, kind: 'success'|'failure'}): Promise<void> }` + `ConsoleNotifySink` (logs info). The osascript sink and message templates are issue 25; this issue calls the seam with a `RunOutcome`-derived payload via a `renderNotification` placeholder that issue 25 finalizes (v1 of the renderer here may emit the §14.3 strings directly — issue 25 owns their tests).
3. `RunOutcome` (explicit contract, consumed by the CLI mapping, issue 25, and e2e):
   ```ts
   type RunOutcome = {
     runId: string;
     status: 'success' | 'partial' | 'failed' | 'no_articles' | 'skipped_existing';
     digest?: { digestId: string; outputPath: string; durationMs: number; articleCount: number };
     failures: RunArticleFailure[];              // from issue 23
     summary: RunSummary;                        // §14.2 shape
     plan?: DryRunPlan;                          // only when dryRun
     error?: { code: string; message: string };  // only when failed
   }
   ```
4. `runPipeline(deps, opts) → RunOutcome` — `deps` injects `{db, cfg, logger, providers: {translationFactory, ttsFactory}, notify, now}`; `opts = {dryRun, force, limit, trigger}`. Phases (§11.1), timings collected per phase into `summary.timings`:
   1. `runId = newId()` generated **before** anything (all orchestrator logs carry `{runId, phase}`; pre-lock failures are still attributable)
   2. acquire lock (dry-run locks too — it performs real ingest); prune logs (`pruneLogs`, §14.1) and old work dirs (`pruneWorkDirs(cacheDir, 7)` — never the current run's dir, which doesn't exist yet)
   3. `startRun({id: runId, trigger})`
   4. ingest (issue 08) — its report goes into the summary (ingest inserts are real state changes even in dry-run, by design — DESIGN §3)
   5. idempotency (issue 23): exists && !force → finish with `RunOutcome.status = 'skipped_existing'` (exit 0; log info; sink never called). Persisted run row uses status `success` with `summary.skippedExisting = true` — the `runs.status` CHECK set (issue 03) is not widened; `skipped_existing` is an in-memory outcome only
   6. select batch (limit override or config)
   7. per article, sequentially: fetch → extract → speechify(placeholderLang=outputLanguage) → detect → `needsTranslation` ? `translateDocument` : passthrough. Results are held **in memory** (`PreparedArticle[]`). Stage throw → `classifyStageError` (issue 23): `article` → `planArticleFailure(article, err)` (pure, issue 23) appended to in-memory `stagedFailures: RunArticleFailure[]`, continue; `run` → jump to step 13 fail path. **No DB writes in this phase** — content persistence and failure accounting are deferred (crash/run-scoped abort must leave articles untouched).
   8. `stats = buildQueueStats(db, stagedFailures)` (issue 23; computed against would-be state)
   9. dry-run → build `DryRunPlan` and finish (see req 5)
   10. prepared == 0 → **terminal commit (empty)**: persist each staged failure via `applyArticleFailure`; finish `no_articles` (exit 0; notify only if failures occurred or `notifyOnEmpty`)
   11. script = `VerbatimScriptGenerator.generate({date, lang: outputLanguage, articles, stats})`; TTS: provider = `ttsFactory(outputLanguage)`; `checkAvailability` fail → run-scoped fail path; `prepare()` (VOICEVOX engine autostart happens inside the provider per issue 18); split utterances per chapter (issue 16) with mapping `{chapterIndex, wavPath, paragraphBreakAfter}`; synthesize sequentially into `<cacheDir>/work/<runId>/`
   12. assemble (issue 22) with `{date, sequence}` → `AssembleResult` → **terminal commit (digest)**: single flow = `createDigestWithItems` transaction (digest + items with chapter times for article chapters positions 1..N + markDigested) followed by `updateContent` per digested article (title/detected_language/content_text/translated_text/word_count) and `applyArticleFailure` per staged failure; then delete the run's work dir (old-workdir pruning uses `pruneWorkDirs` **exported by issue 22's `audio/assemble.ts`**, called in step 2)
   13. finish: status `success` (no failures) / `partial` (≥1 staged failure) with full §14.2 summary; run-scoped failure path: persist **nothing** for articles (staged failures discarded — retry counts must be unchanged), finish `failed` with `error`, keep work dir, notify failure; exit 1
   14. `finally`: provider `dispose()` (stops a provider-started engine), release lock.
5. Dry-run semantics (DESIGN §3 — mutation-free except ingest):
   - executes fetch/extract/speechify/detect in memory; **skips translation** (network/model cost) and TTS/assembly entirely; cross-language articles are marked `translation: 'pending'` in the plan with reading-time estimated from source text (flagged approximate)
   - NO `updateContent`, NO `applyArticleFailure`, NO digest/status writes; run row is recorded with `summary.dryRun = true`, status `success`
   - `DryRunPlan = { articles: Array<{id, title, lang, needsTranslation, estimatedMinutes, error?: {code}}>, estimatedTotalMinutes, wouldRollOver: number }`; stage errors appear as `error` entries (not persisted)
   - human output: one line per article `#N <id8> [lang→out] ~Xmin <title>` + errors marked `!`, footer with totals; `run --dry-run --json` prints the `RunOutcome` (incl. plan) as JSON.
6. `earmark run` flags per DESIGN §3 (`--trigger` hidden, default `cli`); exit-code mapping from `RunOutcome.status`: success/partial/no_articles/skipped_existing → 0; failed → 1; `LOCK_HELD` → 4.
7. Crash-consistency: run row may remain `running` after SIGKILL (accepted, §5.1); the digest transaction is atomic; because failure accounting is deferred to terminal commits, a crash mid-run changes nothing except the run row and work dir.
8. Notification payloads (§14.3 strings) emitted for: success, partial, failed; `no_articles` only per `notifyOnEmpty` or failures present. Exact rendering tests live in issue 25; this issue asserts the seam is called with the right `kind` and outcome data.

## Acceptance Criteria

- [ ] Happy path (fakes: passthrough translation factory, `FakeTtsProvider` from issue 16, assembly fake returning planned times): 3 queued → digest row + 3 items + articles `digested` with content persisted + run `success` + all timings present.
- [ ] Isolation: article 2 of 3 throws `FETCH_HTTP_404` → digest with 2 article chapters; article 2 `retry_count` 1 (persisted at terminal commit) still `queued`; run `partial`; generator received stats with `failedThisRun` length 1 (captured input).
- [ ] Run-scoped: TTS `checkAvailability` false → run `failed`; **all articles byte-identical to pre-run state** (statuses, retry counts, content columns — full row diff), staged failures discarded even though article 2 had failed fetch earlier in the same run; lock released; notify called with `kind:'failure'`.
- [ ] en-path: config `outputLanguage='en'` with a fake en provider → generator called with `lang:'en'`, pipeline completes (proves the ttsFactory language dispatch).
- [ ] Idempotency: second run same day → `RunOutcome.status 'skipped_existing'`, exit 0, no digest, run row persisted as `success` + `summary.skippedExisting`; `--force` → sequence 2.
- [ ] Dry-run: DB diff after run = ingest inserts + run row only (snapshot compare); plan JSON matches the contract incl. `translation: 'pending'` for a cross-language article; no translation provider call (spy).
- [ ] no_articles path A — genuinely empty queue after ingest: exit 0; `notifyOnEmpty:false` suppresses notification; no article rows touched.
- [ ] no_articles path B — batch selected but every article fails article-scoped stages: exit 0; staged failures ARE persisted (retry counts advanced); notification carries the failure count.
- [ ] Notification seam matrix: `success` and `partial` call notify with `kind:'success'`; `failed` with `kind:'failure'`; `skipped_existing` never calls the sink (spy asserts).
- [ ] Lock: concurrent second invocation exits 4; stale lock (dead pid fixture) stolen with warning.
- [ ] Crash simulation: throw between synthesis and commit → no digest rows, no article changes, work dir preserved; `pruneWorkDirs` later removes only dirs older than 7 days.
- [ ] runId present in all captured log lines across phases.

## Validation

`vitest` orchestration suite with injected fakes (table the scenarios). Manual: full real run on the dev Mac (real VOICEVOX + Ollama + 3 real articles incl. 1 English) — wave-4 gate; attach `earmark log --last --json` output to the PR.

## Dependencies

04 (CLI/error mapping), 08, 10, 11, 12, 13, 14, 15, 16, 17, 18, 21, 22, 23 (19/20 exercised via the en-path fake here; real en providers covered in their issues and e2e scenario D).

## Non-goals

osascript notification implementation & `earmark log` (25); launchd installation (26); parallel article processing; resuming a partially-synthesized run (work dir is disposable); persisting successful-stage content for articles that later fail (refetch next run — DESIGN §5.2 note).

## Design References

DESIGN §11.1 (incl. deferred failure persistence note), §11.4, §11.5, §12, §14.2–14.3, §5.1–5.2, §3 (dry-run contract); issues 16/21/22/23 models.
