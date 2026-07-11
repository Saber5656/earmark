# Batch selection & article retry/state policy

## Summary

Implement `src/pipeline/select.ts`: morning batch selection (oldest-first with cap), the run-level retry/failure policy that decides between article-scoped and run-scoped failures, queue statistics for the closing narration, and digest-date idempotency helpers.

## Context

DESIGN §11.2/§11.5 fix selection and idempotency; §12.1 fixes the failure classification the orchestrator (issue 24) applies. Putting the policy in a pure module keeps issue 24 as wiring and makes the rules unit-testable against the state machine (issue 03).

## Scope

- `src/pipeline/select.ts`, `src/pipeline/errors.ts` (classification), tests with a temp DB.

## Detailed Requirements

1. `selectBatch(db, limit) → Article[]`: delegates to repo `selectQueuedBatch(limit)` (added in issue 03); limit = `--limit` override or `digest.maxArticles`.
2. Failure classification (`pipeline/errors.ts`): `classifyStageError(err) → 'article' | 'run'` by `EarmarkError.code` prefix table (DESIGN §12.1):
   - article-scoped: `URL_*`, `FETCH_*`, `EXTRACT_*`, `TRANSLATE_*`
   - run-scoped: `TTS_UNAVAILABLE`, `TTS_SYNTH_FAILED`, `ASSEMBLE_FAILED`, `DB_*`, `CONFIG_*`, unknown/unexpected errors
   - table is exhaustive over §12.2 codes; unknown `EARMARK_*` code → `run` (fail safe: don't burn an article's retries on infrastructure problems).
3. `applyArticleFailure(db, articleId, err) → {status, retryCount, willRetry}`: wraps repo `recordFailure` with retry limit 3, logs warn `{articleId, code, retryCount, willRetry}`; returns data for run summary + closing narration.
4. `buildQueueStats(db, runFailures) → QueueStats` (issue 21 input shape): `failedThisRun` = articles that failed this run but remain `queued`; `permanentlyFailedThisRun` = flipped to `failed` this run; `queueRemaining` = queued count **after** this run's transitions (i.e. still-queued articles not included in the digest).
5. Idempotency helpers (`§11.5`): `digestPlanForToday(db, {force, now}) → {date, sequence} | {alreadyExists: {date, sequence}}` — date = local `YYYY-MM-DD` from injected `now`; without force and existing sequence ≥ 1 → alreadyExists; with force → next sequence.
6. Local date derivation: `Intl.DateTimeFormat('en-CA', {timeZone: undefined, dateStyle: 'short'})`-based or manual from `new Date(now)` local getters — implement `localDateString(now: Date): string` here, the ONLY place local-date logic lives; injected `now` everywhere (no `Date.now()` in module bodies).
7. Everything takes the db handle; no config/CLI imports (layering).

## Acceptance Criteria

- [ ] Selection: seeded queue of 12 with interleaved statuses → batch of 10 oldest queued, `added_at ASC, id ASC` order proven with equal-timestamp rows.
- [ ] Classification table test covers every §12.2 code plus an unknown code (→ run) and a plain `Error` (→ run).
- [ ] `applyArticleFailure` ×3 on the same article: willRetry true/true/false; status queued/queued/failed; retryCount 1/2/3.
- [ ] `buildQueueStats` on a synthetic run (2 soft-failed, 1 permanent, 4 untouched queued, 10 digested) returns exactly {2 titles, 1 title, 4}.
- [ ] Idempotency: no digest → {date, seq 1}; existing seq 1 without force → alreadyExists; with force → seq 2; date boundary test with injected `now` at 23:59:59 vs 00:00:01 local.
- [ ] No `Date.now()`/`new Date()` without injection (lint/grep test).

## Validation

`vitest` with temp DB. No manual steps.

## Dependencies

03.

## Non-goals

Orchestration/wiring (24); time-based caps or newest-first ordering (P10 fixed oldest-first; v2 config could add ordering); replacing failed articles within the same run (DESIGN §11.2 explicitly not).

## Design References

DESIGN §11.2, §11.5, §12.1–12.2, §5.2; issue 21 (QueueStats consumer).
