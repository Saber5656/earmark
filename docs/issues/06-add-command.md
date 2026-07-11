# `earmark add` command

## Summary

Implement `earmark add <url> [--title <t>] [--note <n>]`: validate, normalize, dedupe against the queue, insert as `queued` with `source='cli'`, and print the result.

## Context

This is the Mac-side capture channel (DESIGN §1.1 step 2, §3). It composes issue 05 (validation/normalization) with issue 03 (repository) inside the CLI skeleton (issue 04).

## Scope

- `src/cli/commands/add.ts` + registration in `src/cli/index.ts`; tests.

## Detailed Requirements

1. Signature: `earmark add <url>` with options `--title <string>` (≤ 500 chars, else usage error 2), `--note <string>` (≤ 2000 chars). Exactly one positional URL (commander enforces; extra args → exit 2).
2. Flow: `validateCaptureUrl` → invalid: stderr `error (EARMARK_URL_INVALID): <reason>` exit 2 (user input error). Valid → `normalizeUrl` → `insertArticle({url: raw-as-given-trimmed, normalizedUrl, title, note, source:'cli'})`.
3. Duplicate handling (repository returns existing row): print to stdout `already exists: <id> (status: <status>, added: <YYYY-MM-DD>)` and hint `use: earmark requeue <id>` when status is `failed|digested|archived`; **exit 0** (idempotent capture — safe for shell aliases and Shortcuts-over-ssh later).
4. Success output (stdout, single line): `queued: <id>  <title-or-url>` where title falls back to the raw URL when absent. With global `--json`: `{"result":"queued"|"duplicate","article":{id,url,normalizedUrl,status,title,addedAt}}` and nothing else on stdout.
5. Title/note are stored verbatim except: strip control characters (reuse logger sanitization util from issue 04 — export it from `core/logger.ts` as `stripControl(s)`), trim, collapse internal newlines in title to spaces.
6. No fetching, no network at add time (capture stays instant; content work happens in the morning run).

## Acceptance Criteria

- [ ] `earmark add https://example.com/a?utm_source=x` then `earmark add https://example.com/a` → second prints `already exists` with the first id, exit 0, and only one row exists.
- [ ] `earmark add javascript:alert(1)` → exit 2, `EARMARK_URL_INVALID`, no row.
- [ ] `--title` with embedded `\x1b[31m` stores stripped text.
- [ ] `--json` output is valid JSON matching the shape above in both queued and duplicate cases.
- [ ] DB row: `source='cli'`, `status='queued'`, `retry_count=0`, timestamps ISO-8601 UTC.

## Validation

Unit tests drive the command action with a temp DB + temp config dir (test harness helper from issue 04 pattern). Manual transcript in PR: add / duplicate-add / invalid-add with `echo $?`.

## Dependencies

03, 04, 05.

## Non-goals

Inbox ingestion (08); interactive prompts; batch add from file (v2 if ever); fetching titles from the web at capture time.

## Design References

DESIGN §3 (command table), §7.1–7.2, §5.2 (add transition), §12.4.
