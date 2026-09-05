# iCloud inbox ingest module and `earmark ingest`

## Summary

Implement `src/capture/inbox.ts` and the `earmark ingest` command: scan the iCloud Drive inbox folder for capture files written by the iOS Shortcut, validate them against the DESIGN §6.1 contract, enqueue articles, and archive the files into `processed/` (invalid → `rejected/`). Handle iCloud placeholder (`.icloud`) materialization.

## Context

The iPhone Share Sheet is the primary capture channel (ADR-005). The Shortcut drops one small JSON file per shared URL into `<icloudRoot>/inbox/`; the Mac ingests at morning-run time and on demand. The inbox is a semi-trusted boundary (DESIGN §13.1 B3) — validation must be strict and failures must never crash a run.

## Scope

- `src/capture/inbox.ts` (scan/parse/enqueue/archive/retention), `src/cli/commands/ingest.ts`; tests with fixture directories.

## Detailed Requirements

1. Directory bootstrap: `ensureInboxDirs(cfg)` creates `inbox/`, `inbox/processed/`, `inbox/rejected/` under `resolvedInboxDir(cfg)`'s parent contract (inbox dir itself is the resolved dir; processed/rejected are subdirectories of it).
2. Scan (`scanInbox`): `readdir` of inbox dir only (no recursion); for each entry `lstat`; consider only **regular files** (symlinks, dirs, sockets skipped with a debug log); skip `processed/`, `rejected/`; classify:
   - `*.json` / `*.txt` → candidates
   - `.<name>.icloud` → placeholder set
   - anything else (e.g. `.DS_Store`) → ignore silently.
3. Placeholder materialization: for each placeholder, run `execFile('brctl', ['download', <inboxDir>/<name>])` best-effort (ENOENT of brctl → debug log, skip); after issuing all, poll every 1 s up to **15 s total** for the real files to appear; still-missing files are reported as `pendingDownload` (ingest of them happens next run). Never fail ingest because brctl fails.
4. Parse per file (size check first: `> 65536` bytes → reject `too_large` without reading fully):
   - `.json`: UTF-8 parse → zod schema `{v: literal(1), url: string, title?: string ≤ 500, sharedAt?: string}` with `.passthrough()` ignoring extra keys → then `validateCaptureUrl(url)`
   - `.txt`: first non-empty line trimmed must pass `validateCaptureUrl`; title absent
   - any failure → reason string (`bad_json`, `bad_schema`, `bad_url:<reason>`, `too_large`, `unreadable`).
5. Enqueue valid entries: `validateCaptureUrl(url)` yields the URL → `articleRepo.insert({url: trimmedUrl, normalizedUrl: normalizeUrl(url), title: sanitizedTitle, source: 'ios'})` where `sanitizedTitle` = `stripControl` (issue 04 `core/text.ts`) → newline runs to spaces → trim → truncate 500. A `{duplicate}` result (whether the twin file arrived in this scan or the URL already existed in the DB from any earlier capture) increments `duplicates`, does not appear in `imported`, leaves the existing row untouched — and the file is still archived to `processed/`.
6. File archival (never delete user files):
   - valid & enqueued/duplicate → move to `processed/<sanitized-basename>`; sanitize = basename only, strip control chars, replace path separators; on name collision append `-1`, `-2`, …
   - invalid → move to `rejected/<sanitized-basename>` (same collision rule) and log warn `INBOX_INVALID {file, reason}`
   - moves use `fs.rename`; cross-device fallback copy+unlink not needed (same volume) but EXDEV must produce a warn, not a crash.
7. Retention: after scan, delete files in `processed/` older than `inbox.processedRetentionDays` (by mtime); `rejected/` is **never** auto-pruned (DESIGN §6).
8. Return `IngestReport = {imported: ArticleRef[], duplicates: number, rejected: {file: string, reason: string}[], pendingDownload: number, prunedProcessed: number}` where `ArticleRef = {id: string, url: string, title: string | null}`. Every `file` value in the report, human output, and `INBOX_INVALID` log lines is a **display basename**: `path.basename` → `stripControl` → truncate 80 — never an absolute path or raw user-controlled string (DESIGN §13.1 B3/B7/B8).
9. `earmark ingest [--json]`: runs 1–8, prints human summary (`imported 2, duplicates 1, rejected 1 (see inbox/rejected), pending download 0`) or the report as JSON; exit 0 even when some files were rejected (per-file problems are data, not errors); exit 1 only on environmental failure (inbox dir unresolvable/uncreatable/unwritable) in both human and `--json` modes.
10. Testability: `brctl` invocation and the poll clock injected (interface `ExecFileFn`, `sleep`) so tests run instantly without brctl.

## Acceptance Criteria

- [ ] Fixture matrix passes: valid json; valid txt; json with extra keys (accepted); title 501 chars (rejected `bad_schema`); `javascript:` URL (rejected `bad_url`); 70 KB file (rejected `too_large`, content never fully read — verified via injected fs spy or read-cap implementation); binary garbage `.json` (rejected `bad_json`); symlink to a valid json (skipped); subdirectory (skipped); `.DS_Store` (ignored).
- [ ] Valid files end up in `processed/`, invalid in `rejected/`, with collision suffixing proven.
- [ ] Duplicate URL across two inbox files → 1 imported + 1 duplicate, both archived.
- [ ] DB-preseeded duplicate: an article added earlier via `earmark add` (same normalized URL) → ingest reports `imported: 0, duplicates: 1`, file archived to `processed/`, existing row byte-identical.
- [ ] Environmental failure: unwritable inbox parent → exit 1 in human and `--json` modes; a rejected file alone still exits 0.
- [ ] Report `file` fields are sanitized basenames (fixture with a control-char-bearing name asserts the display form).
- [ ] Placeholder flow: fixture `.x.json.icloud` triggers injected brctl call; file "appearing" mid-poll gets ingested; never-appearing counts as `pendingDownload`.
- [ ] Retention deletes only `processed/` files older than the configured days; `rejected/` untouched.
- [ ] Report numbers match the fixture matrix exactly; `--json` output validates.

## Validation

`vitest` with per-test temp dirs simulating the inbox; injected exec/sleep fakes. Manual (dev Mac, real iCloud): drop a JSON file via Files app on iPhone into `earmark/inbox/`, run `earmark ingest`, verify import + `processed/` move; attach transcript to PR (this is also wave-1 gate evidence).

## Dependencies

02, 03, 04, 05.

## Non-goals

The Shortcut recipe itself (issue 09); file watching/daemon mode; ingesting from arbitrary folders (config covers custom inbox path already); parsing HTML/URL files other than the two contract formats.

## Design References

DESIGN §6.1 (contract), §13.1 B3, §13.8 (abuse cases: traversal/symlink/size), §3 (`ingest`); ADR-005.
