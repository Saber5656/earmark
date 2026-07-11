# Batch selection & article retry/state policy

## Summary

Implement `src/pipeline/select.ts`: morning batch selection (oldest-first with cap), the run-level retry/failure policy that decides between article-scoped and run-scoped failures, queue statistics for the closing narration, and digest-date idempotency helpers.

## Context

DESIGN §11.2/§11.5 fix selection and idempotency; §12.1 fixes the failure classification the orchestrator (issue 24) applies. Putting the policy in a pure module keeps issue 24 as wiring and makes the rules unit-testable against the state machine (issue 03).

## Scope

- `src/pipeline/select.ts`, `src/pipeline/errors.ts` (classification), tests with a temp DB.

## Detailed Requirements

1. `selectBatch(db, limit) → Article[]`: delegates to repo `selectQueuedBatch(limit)` (added in issue 03); limit = `--limit` override or `digest.maxArticles`.
2. Failure classification (`pipeline/errors.ts`): `classifyStageError(err) → 'article' | 'run'` over the actual `EarmarkError.code` values (bare strings; the full inventory across issues 02–22):
   - article-scoped: `URL_INVALID`, `FETCH_TIMEOUT`, `FETCH_HTTP_<status>` (prefix match `FETCH_HTTP_`), `FETCH_TOO_LARGE`, `FETCH_UNSUPPORTED_TYPE`, `FETCH_PRIVATE_BLOCKED`, `FETCH_TOO_MANY_REDIRECTS`, `FETCH_NETWORK`, `EXTRACT_EMPTY`, `TRANSLATE_UNAVAILABLE`, `TRANSLATE_INVALID_OUTPUT`
   - run-scoped: `TTS_UNAVAILABLE`, `TTS_SYNTH_FAILED`, `AUDIO_BAD_WAV`, `ASSEMBLE_FAILED`, `DB_NEWER_SCHEMA`, `DB_ILLEGAL_TRANSITION`, `CONFIG_INVALID`, `LOCK_HELD`
   - any other code, any non-`EarmarkError` → `run` (fail safe: don't burn an article's retries on infrastructure problems). `INBOX_INVALID` never reaches this function (it is a per-file ingest log code, not a thrown pipeline error).
   - `articles.last_error` stores the bare `CODE: message` string — no prefix (DESIGN §12.2).
3. `applyArticleFailure(db, logger, articleId, err) → RunArticleFailure`: wraps repo `recordFailure` with retry limit 3; logs warn `{articleId, code, retryCount, willRetry}` via the passed logger.
4. `RunArticleFailure` (exact contract; also consumed by issues 24/25/28):
   ```ts
   type RunArticleFailure = { articleId: string; title: string | null; code: string; message: string;
                              willRetry: boolean; retryCount: number; status: 'queued' | 'failed' };
   ```
5. `buildQueueStats(db, runFailures: RunArticleFailure[]) → QueueStats` (issue 21 input shape): `failedThisRun` = entries with `status === 'queued'` (will retry); `permanentlyFailedThisRun` = entries with `status === 'failed'`; `queueRemaining` = count of `queued` articles after this run's transitions — this **includes** retryable soft-failed articles (they are still queued) and excludes digested ones.
6. Idempotency helpers (§11.5): `digestPlanForToday(db, {force, now}) → {date, sequence} | {alreadyExists: {date, sequence}}` — date = `localDateString(now)`; without force and existing sequence ≥ 1 → alreadyExists; with force → next sequence.
7. `localDateString(now: Date): string` — manual local getters with zero padding: `getFullYear()`, `getMonth()+1`, `getDate()` (no `Intl` — deterministic across ICU builds). The ONLY place local-date logic lives; `now` injected everywhere (no `Date.now()`/argless `new Date()` in this issue's production files — `src/pipeline/select.ts`, `src/pipeline/errors.ts`).
8. Everything takes the db handle; no config/CLI imports (layering).

## Acceptance Criteria

- [ ] Selection: seeded queue of 12 with interleaved statuses → batch of 10 oldest queued, `added_at ASC, id ASC` order proven with equal-timestamp rows.
- [ ] Classification table test covers every code listed in req 2 (both classes), a `FETCH_HTTP_404` prefix case, an unknown code (→ run), and a plain `Error` (→ run).
- [ ] `applyArticleFailure` ×3 on the same article: willRetry true/true/false; status queued/queued/failed; retryCount 1/2/3; returned `RunArticleFailure` fields exact.
- [ ] `buildQueueStats` on a synthetic run (2 soft-failed retryable, 1 permanent, 4 untouched queued, 10 digested) returns `failedThisRun` 2 titles, `permanentlyFailedThisRun` 1 title, `queueRemaining` **6** (4 untouched + 2 retryable).
- [ ] Idempotency: no digest → {date, seq 1}; existing seq 1 without force → alreadyExists; with force → seq 2; date boundary test with injected `now` at 23:59:59 vs 00:00:01 local; `localDateString` zero-padding (2026-01-05).
- [ ] Grep test scoped to `src/pipeline/select.ts` + `src/pipeline/errors.ts`: no `Date.now()` / argless `new Date()`.

## Validation

`vitest` with temp DB. No manual steps.

## Dependencies

03.

## Non-goals

Orchestration/wiring (24); time-based caps or newest-first ordering (P10 fixed oldest-first; v2 config could add ordering); replacing failed articles within the same run (DESIGN §11.2 explicitly not).

## Design References

DESIGN §11.2, §11.5, §12.1–12.2, §5.2; issue 21 (QueueStats consumer).
