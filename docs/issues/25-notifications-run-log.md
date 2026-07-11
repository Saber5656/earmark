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
   - sanitize both strings first: `stripControl`, collapse whitespace, truncate 200 chars
   - failures (non-zero, timeout 5 s) → log warn once per run, never throw (notification is best-effort)
   - `config.notifications.enabled === false` → sink not constructed (run uses console sink).
2. Notification templates (exact, from DESIGN §14.3): success `Digest ready: {n} articles, {m} min` + ` ({f} failed)` when f > 0; failure `Digest failed: {reason}. Run earmark doctor.`; produced by a pure `renderNotification(outcome)` function shared with tests.
3. `earmark log [--last] [--limit <n>] [--json]`:
   - default: table of the last 10 runs — columns `STARTED(local YYYY-MM-DD HH:MM)  TRIGGER  STATUS  ARTICLES(prepared/selected)  DURATION(run wall time)  DIGEST(file name or -)`
   - `--last`: detailed view of the most recent run: status, timings breakdown (each §14.2 timing as `label 12.3s` lines), failed articles with codes and willRetry, digest path, queueRemaining
   - `--json`: raw `runs` rows with `summary_json` parsed inline (array; with `--last` a single object)
   - `running` rows from crashed processes render status as `running (stale?)` when older than 3 h
   - empty history → `no runs recorded`, exit 0.
4. Duration formatting helper: ms → `42s` / `12m 05s` / `1h 02m` (used by log and notification minutes rounding — minutes = `round(durationMs/60000)`, minimum 1 for non-zero).
5. Summary parse failures (corrupt JSON) render as `⚠ summary unreadable` without crashing.

## Acceptance Criteria

- [ ] osascript argv construction spy-tested with hostile body `"; do shell script "..."` and ANSI/newlines — argv array exact, source lines constant, text sanitized.
- [ ] Sink failure (execFile error injected) → warn logged, notify resolves.
- [ ] `renderNotification` matrix: success 0 failed / success 2 failed / failure — exact strings.
- [ ] `log` table snapshot with seeded runs (success/partial/failed/no_articles/stale-running); `--last` detail snapshot; `--json` parses with parsed summaries.
- [ ] Corrupt summary_json row renders with warning marker, exit 0.
- [ ] Darwin integration test (CI): real osascript invocation with `enabled:true` — assert exit 0 (visual check not needed).

## Validation

`vitest` (darwin integration in CI). Manual: trigger a real run on the dev Mac and confirm the notification banner appears; attach screenshot at wave-4 gate.

## Dependencies

24 (RunOutcome/summary shapes; sink wiring point).

## Non-goals

Notification actions/sounds; email/webhook notifications; `log` filtering by status; localizing CLI/notification text (English, consistent with CLI; digest narration is the localized surface).

## Design References

DESIGN §14.2, §14.3, §13.1 B6; issue 24 (summary producer).
