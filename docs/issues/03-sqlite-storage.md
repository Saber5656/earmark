# SQLite storage: schema migration, repositories, state transitions

## Summary

Implement `src/core/db.ts` (better-sqlite3 open + migration runner) and the repository layer `src/core/repo/{articles,digests,runs}.ts` exposing every state transition from DESIGN §5.2 as named functions. All SQL is parameterized; no other module may execute SQL.

## Context

The queue is the heart of earmark: articles move `queued → digested/failed/archived` with retry accounting; digests and runs are recorded for idempotency and `earmark log`. DESIGN §5 fixes the DDL; this issue turns it into code with an enforced single write path.

## Scope

- `src/core/db.ts`: open, PRAGMAs, migration runner, `meta` table
- `src/core/repo/articles.ts`, `repo/digests.ts`, `repo/runs.ts` (repo factory pattern)
- `src/core/ids.ts`: ULID generation
- Runtime deps added: `better-sqlite3@12.11.1` (exact pin per DESIGN §13.7), `ulid@^3`; dev dep `@types/better-sqlite3`

## Detailed Requirements

1. `openDb(cfg)`: ensure `paths.dataDir` exists; open `<dataDir>/earmark.db`; PRAGMAs per connection: `journal_mode = WAL`, `foreign_keys = ON`, `busy_timeout = 5000`. Run migrations; return `EarmarkDb` (thin typed wrapper holding the `Database`).
2. Migration runner and bootstrap algorithm (explicit):
   - current version = `meta` table missing → 0; else `SELECT value FROM meta WHERE key='schema_version'` as integer
   - migrations are an ordered in-code array `[{version: 1, up(db)}]`; each pending migration runs inside a single transaction that also creates/updates the `schema_version` row (migration 001 itself creates the `meta` table first, then all §5.1 objects, then inserts `schema_version = 1`)
   - DB version **newer** than the code's max → `EarmarkError DB_NEWER_SCHEMA` with exit code 3 (invariant/config class per DESIGN §12.1) telling the user to upgrade earmark.
3. Migration 001 creates exactly the DDL of DESIGN §5.1 (tables `articles`, `digests`, `digest_items`, `runs`; all indexes including `idx_digest_items_article` UNIQUE on `(digest_id, article_id)`; CHECK and UNIQUE constraints).
4. Repository pattern: `createArticleRepo(db): ArticleRepo` (likewise digests/runs) — prepared statements created once inside the factory; all methods synchronous; timestamps ISO-8601 UTC generated via an injectable `now(): Date` (defaults to `new Date`) for testability.
5. `ArticleRepo` methods:
   - `insert({url, normalizedUrl, title?, note?, source}) → {created: Article} | {duplicate: Article}` (status `queued`, retry_count 0; UNIQUE violation on `normalized_url` returns the existing row unchanged)
   - `findById(id)`, `findByNormalizedUrl(nurl)`, `findByIdPrefix(prefix)` (≥ 8 chars; `{match} | {ambiguous: Article[]} | {none}`)
   - `listByStatus(status | 'all', limit, offset=0)` ordered `added_at DESC, id DESC`
   - `selectQueuedBatch(limit)` ordered `added_at ASC, id ASC` (DESIGN §11.2)
   - transitions (validate current status per DESIGN §5.2, else throw `EarmarkError DB_ILLEGAL_TRANSITION`): `markDigested(id, digestedAt)`, `recordFailure(id, code, message, retryLimit=3)` → `{status: 'queued'|'failed', retryCount}` (increments; flips to `failed` at limit; `last_error = "CODE: message"` truncated 500 chars), `archive(id)` (from `queued` only), `requeue(id)` (from `failed|digested|archived`; resets retry_count 0, clears last_error)
   - `updateContent(id, {title?, detectedLanguage?, contentText?, translatedText?, wordCount?})`
   - `queueStats() → {queued, failed, digested, archived}`
   - every mutation updates `updated_at`.
6. `DigestRepo`:
   - `existsFor(date) → maxSequence | 0`
   - `createDigestWithItems(input: DigestCommit)` where
     ```ts
     type DigestCommit = {
       digest: { id: string; digestDate: string; sequence: number; outputPath: string; durationMs: number; createdAt: string };
       items: Array<{ articleId: string; position: number; chapterTitle: string; startMs: number; endMs: number }>;
     }
     ```
     Single transaction: insert digest with `article_count = items.length`; insert all digest_items; `markDigested(articleId, digest.createdAt)` for each. Invariants enforced before any write: positions are exactly `1..items.length` contiguous; `articleId`s unique (duplicate → throw, nothing written; the UNIQUE index is the DB-level backstop); every article currently `queued`. Any failure rolls back everything.
7. `RunRepo`: `startRun({id, trigger, startedAt}) → void` (status `running`; id supplied by caller so logs can carry it beforehand), `finishRun(id, status, {digestId?, summaryJson})`, `latestRuns(limit)`. Crashed `running` rows are surfaced as-is by `latestRuns` (display concerns are issue 25's).
8. Error-code note: `DB_NEWER_SCHEMA` / `DB_ILLEGAL_TRANSITION` are internal `EarmarkError.code` values for CLI error handling; they are **never** written to `articles.last_error` (that column only receives pipeline codes per DESIGN §12.2).
9. `ids.ts`: `newId()` → ULID (monotonic factory).

## Acceptance Criteria

- [ ] Fresh open creates the DB with schema_version 1 and all DESIGN §5.1 objects incl. `idx_digest_items_article` (verified via `sqlite_master`).
- [ ] Per-connection PRAGMAs asserted: `journal_mode` = `wal`, `foreign_keys` = 1, `busy_timeout` = 5000.
- [ ] Full transition table of DESIGN §5.2 covered, including every illegal transition throwing `DB_ILLEGAL_TRANSITION`.
- [ ] `recordFailure` flips to `failed` exactly on the 3rd failure; `requeue` resets and allows 3 more.
- [ ] Duplicate `insert` returns `{duplicate}` with the original row unmodified.
- [ ] `createDigestWithItems` atomicity matrix: (a) mid-transaction injected error → no digest/digest_items rows, articles still `queued`; (b) duplicate articleId in items → throw before any write; (c) non-contiguous positions → throw; (d) article not `queued` → throw and rollback.
- [ ] `selectQueuedBatch` ordering: older `added_at` first; ULID ascending tiebreak.
- [ ] DB with `schema_version = 999` → `DB_NEWER_SCHEMA` with exit code 3 mapping.

## Validation

`vitest` suite using a temp data dir per test, plus a WAL smoke test (two sequential connections). Attach `npm test` output to the PR.

## Dependencies

01, 02 (`EarmarkError` from `core/errors.ts`, config types for `paths.dataDir`).

## Non-goals

Business policy (which stage failures call `recordFailure` — issues 23/24); CLI commands (06/07); backup/vacuum tooling; multi-process write coordination beyond `busy_timeout` (the run lock in issue 24 serializes writers).

## Design References

DESIGN §5 (data model incl. the digest_items uniqueness invariant), §11.2, §12.1–12.2; ADR-002.
