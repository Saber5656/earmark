# ADR-005: iCloud Drive folder as the iPhone→Mac capture transport

- Status: Accepted (2026-07-11)
- Deciders: product owner (capture channels: CLI + iPhone Share Sheet), Fable (transport detail)

## Context

The owner captures most articles on iPhone. earmark runs only on the Mac and must receive URLs shared from iOS with minimal moving parts, no server, and no accounts (ADR-001). Apple's supported automation path from the Share Sheet is the Shortcuts app.

## Decision

1. An iOS Shortcut ("Save to earmark", recipe in `docs/SHORTCUT.md`) writes **one small JSON file per shared URL** into `iCloud Drive/earmark/inbox/` (contract in DESIGN §6.1: `{v:1, url, title?, sharedAt}` ≤ 64 KiB; `*.txt` with a URL as a tolerated fallback).
2. The Mac ingests at run time (and via `earmark ingest`): validate → enqueue → move file to `inbox/processed/` (invalid → `inbox/rejected/`). No file watcher daemon in v1; ingest is pull-based at 06:00 or on demand.
3. iCloud placeholder files (`.<name>.icloud`) are materialized best-effort with `brctl download`, waiting ≤ 15 s, else deferred to the next run.

## Consequences

- Zero infrastructure: no server, no push service, no pairing; works anywhere the user's iCloud works.
- Latency: capture→queue is bounded by iCloud sync; a URL shared minutes before 06:00 may miss that morning (accepted; it rolls into tomorrow).
- Inbox is a semi-trusted boundary (any device on the Apple ID writes there) → strict validation + quarantine (DESIGN §13.1 B3).
- Headless-sync behavior of iCloud under launchd is a **known unknown** (ISSUE_PLAN) — mitigated by `brctl` trigger, deferral logic, and doctor checks.

## Alternatives considered

- **Local HTTP endpoint + Shortcut "Get contents of URL"**: instant delivery but requires earmark to listen on a socket (violates no-listener posture), Mac reachability from the phone, and auth design — rejected for v1 (recorded v2 idea alongside a bookmarklet).
- **Shortcuts → SSH to Mac**: requires enabling Remote Login and key management on the phone — rejected (secret-adjacent, fragile).
- **Apple Notes/Reminders as queue, read via AppleScript**: fragile scraping of app internals, permission prompts — rejected.
- **Push via third-party service (ntfy, webhook relays)**: cloud dependency and privacy leak — rejected.
