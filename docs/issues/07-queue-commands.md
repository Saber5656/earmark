# Queue commands: `list`, `remove`, `requeue`

## Summary

Implement queue management: `earmark list` (status-filtered table / JSON), `earmark remove <id>` (archive), `earmark requeue <id>` (reset back to queued). Includes unique-prefix id resolution shared by all id-taking commands.

## Context

Users inspect the backlog, drop mistakes, and resurrect failed/heard articles (DESIGN §3, §5.2). Failed articles surface here with their `last_error` so the morning digest's "1 article failed" line is actionable.

## Scope

- `src/cli/commands/list.ts` (registers `list`, `remove`, `requeue`); id-resolution helper in `src/core/repo/articles.ts` already exists (issue 03 `findByIdPrefix`); tests.

## Detailed Requirements

1. `earmark list [--status queued|digested|failed|archived|all] [--limit <n>] [--json]`:
   - default `--status queued`, `--limit 50` (1–500 else exit 2)
   - ordering (all statuses incl. `all`): `added_at DESC, id DESC` (repo `listByStatus`, issue 03) — deterministic for snapshots
   - human output — exact layout (2-space column gap; TITLE truncated at 60 code points with `…`; null title → the article's `url`; date column = `added_at.slice(0,10)` except `digested` rows show `digested_at.slice(0,10)`):
     ```
     ID        DATE        STATUS    TITLE
     01j8x2a9  2026-07-10  queued    TypeScript 5.9の新機能まとめ…
     01j8wz01  2026-07-09  failed    Understanding launchd
               └ FETCH_HTTP_403: forbidden
     2 shown / queued 1, failed 1, digested 0, archived 0
     ```
     (`failed` rows get the indented `└ <last_error ≤ 100 chars>` second line)
   - empty result → `no articles (status: <s>)`, exit 0
   - `--json`: `{"articles": ArticleRow[], "stats": {queued, failed, digested, archived}}` where `ArticleRow` = all columns except `content_text`/`translated_text` (ISO strings as-is, nulls as null); the only stdout output.
2. Id resolution for `remove`/`requeue`: accept full ULID or a prefix (case-insensitive, ≥ 8 chars; shorter → exit 2 `prefix too short`). `none` → exit 1 `error (NOT_FOUND)` (CLI-only code per issue 04 req 6 — never stored in `last_error`); `ambiguous` → exit 1 listing candidate ids+titles (max 5).
3. `earmark remove <id>`: allowed only from `queued` (DESIGN §5.2) → `archived`; success prints `archived: <id>  <title>`; wrong current status → exit 1 with message stating the actual status and the allowed operation (`requeue`? `remove` only from queued).
4. `earmark requeue <id>`: allowed from `failed|digested|archived` → `queued` with retry_count 0, last_error cleared; success prints `requeued: <id>  <title>`; requeue of a `queued` article → exit 1 `already queued`.
5. Both mutations log at info level with `{articleId, from, to}`.
6. Column truncation must be Unicode-safe (truncate by code points, append `…`); never emit raw control chars — apply `stripControl` (issue 04 `core/text.ts`: ANSI/C0/C1/zero-width/bidi) to every printed field as defense in depth even though capture already sanitizes.

## Acceptance Criteria

- [ ] Seeded DB (fixtures: 3 queued incl. one with a 120-char ja title, 1 failed with last_error, 1 digested, 1 archived; injected fixed timestamps) renders the documented layout byte-exact (snapshot) with correct ordering and footer counts.
- [ ] `--status all` shows every row in `added_at DESC, id DESC` order; `--json` parses as `{articles, stats}`.
- [ ] `remove` on queued works; on digested exits 1 without change.
- [ ] `requeue` matrix: from `failed` (resets retry_count, clears last_error — DB verified), from `digested`, from `archived` → each lands `queued`; on already-`queued` → exit 1 `already queued`, row unchanged.
- [ ] 8-char unique prefix resolves; ambiguous prefix lists ≤5 candidates and exits 1; 7-char prefix → exit 2 (`prefix too short`).
- [ ] A stored title containing raw ESC (injected directly via repo in the test) prints stripped.
- [ ] Exit codes: 0 on success/empty list; 1 not-found/illegal-state; 2 bad flags.

## Validation

Unit tests with temp DB fixtures incl. snapshot of human output (strip dates via injection of a fixed clock helper if needed — repo functions accept `now` injection? If not, snapshot with regex placeholders). Manual transcript: full add→list→remove→requeue cycle.

## Dependencies

03, 04.

## Non-goals

Pagination UX beyond `--limit`; editing title/note; bulk operations; showing article body text.

## Design References

DESIGN §3, §5.2 (transitions incl. requeue semantics), §12.3, §12.4.
