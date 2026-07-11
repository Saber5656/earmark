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
   - human output: aligned columns `ID(8-char prefix)  ADDED(YYYY-MM-DD)  STATUS  TITLE(≤60 chars, ellipsis)`; for `failed` rows append a second indented line `└ <last_error ≤ 100 chars>`; for `digested` rows show `digested_at` date instead of added
   - footer line: `<n> shown / queued <q>, failed <f>, digested <d>, archived <a>` (from `queueStats()`)
   - empty result → `no articles (status: <s>)`, exit 0
   - `--json`: array of full article rows (all columns except `content_text`/`translated_text`), plus `stats` object; nothing else on stdout.
2. Id resolution for `remove`/`requeue`: accept full ULID or a prefix (case-insensitive, ≥ 8 chars). `none` → exit 1 `error (EARMARK_NOT_FOUND)`; `ambiguous` → exit 1 listing the candidate ids+titles (max 5).
3. `earmark remove <id>`: allowed only from `queued` (DESIGN §5.2) → `archived`; success prints `archived: <id>  <title>`; wrong current status → exit 1 with message stating the actual status and the allowed operation (`requeue`? `remove` only from queued).
4. `earmark requeue <id>`: allowed from `failed|digested|archived` → `queued` with retry_count 0, last_error cleared; success prints `requeued: <id>  <title>`; requeue of a `queued` article → exit 1 `already queued`.
5. Both mutations log at info level with `{articleId, from, to}`.
6. Column truncation must be Unicode-safe (truncate by code points, append `…`); never emit raw control chars (repo data is already stripped at capture, but apply `stripControl` on output as defense in depth).

## Acceptance Criteria

- [ ] Seeded DB (fixtures: 3 queued incl. one with 120-char ja title, 1 failed with last_error, 1 digested, 1 archived) renders the exact documented layout (snapshot test) and correct footer counts.
- [ ] `--status all` shows every row; `--json` parses and includes `stats`.
- [ ] `remove` on queued works; on digested exits 1 without change; `requeue` on failed resets retry_count and clears last_error (verified in DB).
- [ ] 8-char unique prefix resolves; 8-char ambiguous prefix lists candidates and exits 1; 7-char prefix → exit 2 usage error (`prefix too short`).
- [ ] Exit codes: 0 on success/empty list; 1 not-found/illegal-state; 2 bad flags.

## Validation

Unit tests with temp DB fixtures incl. snapshot of human output (strip dates via injection of a fixed clock helper if needed — repo functions accept `now` injection? If not, snapshot with regex placeholders). Manual transcript: full add→list→remove→requeue cycle.

## Dependencies

03, 04.

## Non-goals

Pagination UX beyond `--limit`; editing title/note; bulk operations; showing article body text.

## Design References

DESIGN §3, §5.2 (transitions incl. requeue semantics), §12.3, §12.4.
