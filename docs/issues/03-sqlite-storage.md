# SQLite storage: schema migration, repositories, state transitions

## Summary

Implement `src/core/db.ts` (better-sqlite3 open + migration runner) and the repository layer `src/core/repo/{articles,digests,runs}.ts` exposing every state transition from DESIGN §5.2 as named functions. All SQL is parameterized; no other module may execute SQL.

## Context

The queue is the heart of earmark: articles move `queued → digested/failed/archived` with retry accounting; digests and runs are recorded for idempotency and `earmark log`. DESIGN §5 fixes the DDL; this issue turns it into code with an enforced single write path.

## Scope

- `src/core/db.ts`: open, PRAGMAs, migration runner, `meta` table
- `src/core/repo/articles.ts`, `repo/digests.ts`, `repo/runs.ts`
- `src/core/ids.ts`: ULID generation (dep `ulid@^3`)
- Runtime dep added: `better-sqlite3@12` (exact pin), `ulid@^3`; dev dep `@types/better-sqlite3`

## Detailed Requirements

1. `openDb(cfg)`: ensure `paths.dataDir` exists; open `<dataDir>/earmark.db`; PRAGMAs: `journal_mode = WAL`, `foreign_keys = ON`, `busy_timeout = 5000`. Run migrations; return the `Database` handle (typed wrapper `EarmarkDb`).
2. Migration runner: `meta` table `(key TEXT PRIMARY KEY, value TEXT NOT NULL)`; `schema_version` row; migrations are an ordered in-code array `[{version: 1, up(db)}]`; each runs in a transaction; version recorded after success; opening a DB with a **newer** version than the code knows → `EarmarkError DB_NEWER_SCHEMA` (exit 1) telling the user to upgrade earmark.
3. Migration 001 creates exactly the DDL of DESIGN §5.1 (tables `articles`, `digests`, `digest_items`, `runs`; indexes; CHECK constraints; UNIQUE constraints).
4. `repo/articles.ts` (all functions take the db handle; timestamps ISO-8601 UTC generated inside):
   - `insertArticle({url, normalizedUrl, title?, note?, source}) → Article` (status `queued`, retry_count 0); UNIQUE violation on `normalized_url` → typed result `{duplicate: existing}` rather than throw.
   - `findByNormalizedUrl(nurl)`, `findById(id)`, `findByIdPrefix(prefix)` (≥ 8 chars; returns `{match} | {ambiguous: Article[]} | {none}`).
   - `listByStatus(status | 'all', limit, offset=0)` ordered `added_at DESC` for display.
   - `selectQueuedBatch(limit)` ordered `added_at ASC, id ASC` (DESIGN §11.2).
   - Transitions (each validates the current status per DESIGN §5.2 and throws `EarmarkError DB_ILLEGAL_TRANSITION` otherwise): `markDigested(id, digestedAt)`, `recordFailure(id, code, message, retryLimit=3)` → returns `{status: 'queued'|'failed', retryCount}` (increments; flips to `failed` at limit; stores `last_error = "CODE: message"` truncated 500 chars), `archive(id)` (from `queued` only), `requeue(id)` (from `failed|digested|archived`; resets retry_count 0, clears last_error), `updateContent(id, {title?, detectedLanguage?, contentText?, translatedText?, wordCount?})`.
   - `queueStats()` → `{queued, failed, digested, archived}` counts.
   - Every mutation updates `updated_at`.
5. `repo/digests.ts`: `existsFor(date) → maxSequence | 0`; `createDigestWithItems(digest, items[], articleIds[])` — single transaction: insert digest, insert digest_items, call `markDigested` for each article; any failure rolls back all.
6. `repo/runs.ts`: `startRun(trigger) → runId` (status `running`); `finishRun(runId, status, {digestId?, summaryJson})`; `latestRuns(limit)`; unfinished `running` rows from crashed processes are surfaced by `latestRuns` as-is (doctor/log display them; no auto-repair in v1).
7. No ORM; prepared statements created once per repo instance; article text columns accept up to ~1 MB strings (no artificial limit; callers cap sizes).
8. `ids.ts`: `newId()` → ULID (monotonic factory), lowercase not required (store as produced).

## Acceptance Criteria

- [ ] Fresh open creates the DB with schema_version 1 and all DESIGN §5.1 objects (verified in test via `sqlite_master`).
- [ ] Full transition table of DESIGN §5.2 is covered by tests, including every illegal transition throwing `DB_ILLEGAL_TRANSITION`.
- [ ] `recordFailure` flips to `failed` exactly on the 3rd failure; `requeue` resets and allows 3 more.
- [ ] Duplicate `insertArticle` returns the existing row, does not throw, does not modify it.
- [ ] `createDigestWithItems` is atomic: forced mid-transaction error leaves no digest/digest_items rows and articles still `queued`.
- [ ] `selectQueuedBatch` ordering: older `added_at` first; ULID ascending as tiebreak.
- [ ] Opening a DB with `schema_version = 999` fails with `DB_NEWER_SCHEMA`.

## Validation

`vitest` suite using a temp data dir per test. Include a WAL smoke test (two sequential connections). Run `npm test` and attach output. Manual: `node -e` snippet in the PR description opening a scratch DB and printing `queueStats()`.

## Dependencies

01.

## Non-goals

Business policy (which stage failures call `recordFailure` — issues 23/24); CLI commands (06/07); backup/vacuum tooling; multi-process write coordination beyond `busy_timeout` (the run lock in issue 24 serializes writers).

## Design References

DESIGN §5 (data model, state machine), §11.2, §12.2 (error string format); ADR-002.
