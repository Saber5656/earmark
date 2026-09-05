# macOS notifications & `earmark log`

## Summary

Implement `src/pipeline/notify.ts`'s osascript sink (macOS user notifications) and `src/cli/commands/log.ts` (`earmark log`): human-readable rendering of stored run summaries with `--last`, `--limit`, `--json`.

## Context

The 06:00 run is unattended; the notification is how the user learns the digest is ready or that something needs attention (DESIGN §14.3), and `earmark log` is the debugging window into past runs (§14.2). Notification text includes attacker-influenceable counts only (no titles) — keep it that way.

## Scope

- `OsascriptNotifySink` in `src/pipeline/notify.ts`, `src/cli/commands/log.ts`, wiring the sink into `run` (replace ConsoleNotifySink default when `notifications.enabled` and platform darwin), tests.

## Detailed Requirements

1. `OsascriptNotifySink.notify({title, body, kind})`:
   - command: `execFile('/usr/bin/osascript', ['-e', 'on run argv', '-e', 'display notification (item 2 of argv) with title (item 1 of argv)', '-e', 'end run', title, body])` — text passed as **argv items**, never interpolated into the AppleScript source (injection-proof by construction; boundary B6)
   - sanitize both strings first: `stripControl` (issue 04 `core/text.ts`), collapse whitespace runs, truncate 200 chars
   - failures (non-zero, timeout 5 s) → log warn once per run, never throw (notification is best-effort)
   - `config.notifications.enabled === false` → sink not constructed (run uses console sink).
2. `renderNotification(outcome: RunOutcome) → {title, body, kind}` — pure, shared with tests. `title` is always the fixed string `earmark`; `body` by `outcome.status`:
   - `success`/`partial`: `Digest ready: {n} articles, {m} min` + ` ({f} failed)` when f > 0 — `n` = `outcome.digest.articleCount`, `m` = `max(1, round(outcome.digest.durationMs / 60000))`, `f` = `outcome.failures.length`
   - `failed`: `Digest failed: {reason}. Run earmark doctor.` — `reason` = `outcome.error.code` + `: ` + first 80 chars of `outcome.error.message`
   - `no_articles` (emitted only when `notifyOnEmpty` or failures exist, per issue 24): `No digest this morning: queue is empty.` + ` ({f} failed)` when f > 0
   - `skipped_existing`: no notification (issue 24 never calls the sink for it).
3. `earmark log [--last] [--limit <n>] [--json]`:
   - default: table of the last 10 runs — columns `STARTED(local YYYY-MM-DD HH:MM)  TRIGGER  STATUS  ARTICLES(prepared/selected)  DURATION(run wall time)  DIGEST(file name or -)`
   - `--last`: detailed view of the most recent run: status, timings breakdown (each §14.2 timing as `label 12.3s` lines), failed articles with codes and willRetry, digest path, queueRemaining
   - `--json`: `runs` rows with `summary_json` parsed inline as `summary` (array; with `--last` a single object); a row whose `summary_json` fails to parse gets `summary: null, summaryParseError: true`
   - `running` rows from crashed processes render status as `running (stale?)` when older than 3 h
   - empty history → `no runs recorded`, exit 0.
4. Duration formatting helper: ms → `42s` / `12m 05s` / `1h 02m` (used by log and notification minutes rounding — minutes = `round(durationMs/60000)`, minimum 1 for non-zero).
5. Summary parse failures (corrupt JSON) render as `⚠ summary unreadable` without crashing.

## Acceptance Criteria

- [ ] osascript argv construction spy-tested with hostile body `"; do shell script "..."` and ANSI/newlines — argv array exact, AppleScript source lines constant, text sanitized; `title` argv item is exactly `earmark`.
- [ ] Sink failure (execFile error injected) → warn logged, notify resolves.
- [ ] `renderNotification` matrix — exact strings for: success 0 failed / partial 2 failed / failed (code+message truncation) / no_articles with `notifyOnEmpty` / no_articles with 1 failure.
- [ ] `log` table snapshot with seeded runs (success/partial/failed/no_articles/stale-running); `--last` detail snapshot; `--json` parses with `summary` objects.
- [ ] Corrupt summary_json: human render shows the warning marker; `--json` yields `summary: null, summaryParseError: true`; both exit 0.
- [ ] Darwin integration test (CI): real osascript invocation with `enabled:true` — assert exit 0 (visual check not needed).

## Validation

`vitest` (darwin integration in CI). Manual: trigger a real run on the dev Mac and confirm the notification banner appears; attach screenshot at wave-4 gate.

## Dependencies

24 (RunOutcome/summary shapes; sink wiring point).

## Non-goals

Notification actions/sounds; email/webhook notifications; `log` filtering by status; localizing CLI/notification text (English, consistent with CLI; digest narration is the localized surface).

## Design References

DESIGN §14.2, §14.3, §13.1 B6; issue 24 (summary producer).
