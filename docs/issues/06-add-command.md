# `earmark add` command

## Summary

Implement `earmark add <url> [--title <t>] [--note <n>] [--json]`: validate, normalize, dedupe against the articles table (all statuses), insert as `queued` with `source='cli'`, and print the result.

## Context

This is the Mac-side capture channel (DESIGN §1.1 step 2, §3). It composes issue 05 (validation/normalization) with issue 03 (repository) inside the CLI skeleton (issue 04).

## Scope

- `src/cli/commands/add.ts` + registration in `src/cli/index.ts`; tests.

## Detailed Requirements

1. Signature: `earmark add <url>` with options `--title <string>`, `--note <string>`, `--json` (a command option per issue 04's pattern; DESIGN §3 notes it). Exactly one positional URL (commander enforces; extra positionals → exit 2).
2. Flow: `validateCaptureUrl` → invalid: stderr `error (URL_INVALID): <reason>`, exit 2 (user input error). Valid → `normalizeUrl` → `articleRepo.insert({url: trimmed-raw, normalizedUrl, title, note, source:'cli'})`. DB open failure from a broken better-sqlite3 native binding → exit 1 with remediation hint `try: npm rebuild better-sqlite3` (this command owns that hint per issue 04 req 9).
3. Sanitization (exact order, applied to title and note before length checks): `stripControl` (issue 04 `core/text.ts` — removes ANSI sequences, C0 except `\n`, C1, zero-width, bidi) → title only: replace `\n` runs with a single space → trim. Then length limits: title ≤ 500 chars, note ≤ 2000 chars — exceeding after sanitization → exit 2 usage error. Note keeps its newlines.
4. Duplicate handling (repository returns `{duplicate}`): stdout `already exists: <id> (status: <status>, added: <YYYY-MM-DD>)` where the date is `added_at.slice(0, 10)` (UTC); append hint `use: earmark requeue <id>` when status is `failed|digested|archived`; **exit 0** (idempotent capture — safe for shell aliases and automation).
5. Success output (stdout, single line): `queued: <id>  <title-or-url>` (title falls back to the trimmed raw URL). With `--json`: `{"result":"queued"|"duplicate","article":{id,url,normalizedUrl,status,title,addedAt}}` as the only stdout output.
6. No fetching, no network at add time (capture stays instant; content work happens in the morning run). The command module must not import anything from `src/content/`.

## Acceptance Criteria

- [ ] `earmark add https://example.com/a?utm_source=x` then `earmark add https://example.com/a` → second prints `already exists` with the first id and the `added:` date from `added_at.slice(0,10)`, exit 0, one row total.
- [ ] `earmark add javascript:alert(1)` → exit 2, `URL_INVALID`, no row; extra positional (`add u1 u2`) → exit 2.
- [ ] Length limits: 501-char `--title` and 2001-char `--note` (after sanitization) → exit 2, no row; a title that only exceeds 500 chars **before** stripping ANSI noise is accepted.
- [ ] Sanitization: `--title` with ANSI CSI + newline + bidi stores single-line stripped text; `--note` with newlines stores them (control chars stripped).
- [ ] `--json` output is valid JSON matching the shape above in both queued and duplicate cases.
- [ ] DB row: `source='cli'`, `status='queued'`, `retry_count=0`, timestamps ISO-8601 UTC.
- [ ] No-network guard: test asserts the command module's import graph contains no `src/content/` module (static check on the built file or eslint-restricted import for `cli/commands/add.ts`).

## Validation

Unit tests drive the command action with a temp DB + temp config dir (test harness helper from issue 04 pattern). Manual transcript in PR: add / duplicate-add / invalid-add with `echo $?`.

## Dependencies

03, 04, 05.

## Non-goals

Inbox ingestion (08); interactive prompts; batch add from file (v2 if ever); fetching titles from the web at capture time.

## Design References

DESIGN §3 (command table), §7.1–7.2, §5.2 (add transition), §12.4.
