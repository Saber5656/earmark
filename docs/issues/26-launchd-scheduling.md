# launchd scheduling: `earmark schedule install/uninstall/status`

## Summary

Implement `src/schedule/launchd.ts` and `src/cli/commands/schedule.ts`: generate the LaunchAgent plist, install/uninstall it via `launchctl`, and report status including drift detection (plist pointing at a moved node/earmark or stale hour/minute).

## Context

DESIGN §11.3 fixes the plist contract and the modern `launchctl bootstrap/bootout` flow. launchd environments are minimal (no user PATH, no shell init), so absolute paths and an explicit PATH are mandatory; upgrades that move the node binary or the package must be detectable (`doctor`/`status` drift check).

## Scope

- `src/schedule/launchd.ts` (pure plist rendering + launchctl calls behind an exec seam), `src/cli/commands/schedule.ts`, tests.

## Detailed Requirements

1. Plist rendering `renderPlist(params) → string` (pure; golden-tested): params `{nodePath, cliEntryPath, hour, minute, logDir, configPath?}` — the label is NOT a parameter: module constant `LAUNCHD_LABEL = 'dev.earmark.daily'` used everywhere (render, bootout, bootstrap, print); content exactly:
   - `Label` = `dev.earmark.daily`
   - `ProgramArguments` = `[nodePath, cliEntryPath, 'run', '--trigger', 'launchd']` + (`'--config', configPath` when a non-default config path is active at install time)
   - `StartCalendarInterval` dict `{Hour, Minute}`; `RunAtLoad` false
   - `StandardOutPath` = `<logDir>/launchd.out.log`, `StandardErrorPath` = `<logDir>/launchd.err.log`
   - `EnvironmentVariables` = `{ PATH: "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin" }`
   - proper XML plist 1.0 with **XML-escaped** string values (paths can contain `&`, spaces, unicode).
2. Path resolution at install time: `nodePath = process.execPath`; `cliEntryPath` = the package's built entry (`dist/cli/index.js`) resolved from the running module's location (`import.meta.url` → package root → dist path; works for global npm install and repo checkout — derive, don't hardcode; when running from `tsx` dev context without dist, refuse with message "build first or install the package").
3. `install [--hour H --minute M]`: flags override config `schedule.*` and are **persisted back to config** (single source of truth); write plist to `~/Library/LaunchAgents/dev.earmark.daily.plist` (0644, dir created); then `launchctl bootout gui/<uid>/dev.earmark.daily` (exit-code tolerated) → `launchctl bootstrap gui/<uid> <plist>`; bootstrap failure → error with stderr and hint (`launchctl` errors are cryptic; include "try: launchctl bootout ... first" text). Prints next-fire description `installed: daily at 06:00 (label dev.earmark.daily)`.
4. `uninstall`: bootout (tolerate not-loaded) + delete plist (tolerate missing); prints what happened.
5. `status [--json]` (the `--json` flag on `status` is part of this issue's CLI surface; DESIGN §3 notes it): reports — plist exists?; loaded? (`launchctl print gui/<uid>/dev.earmark.daily` exit 0; parse `state = ` line best-effort); schedule (parsed from plist); drift check: plist's nodePath/cliEntryPath exist on disk AND equal current resolutions AND hour/minute match config → `ok`, else list each mismatch with remediation `re-run: earmark schedule install`; `--json` structured output. Exit 0 always (status is informational; doctor decides severity).
6. uid via `process.getuid()`; all launchctl/exec via injected `ExecFileFn` seam.
7. Missed-run semantics documented in `status` output footer (user education per DESIGN §11.3, honest wording): `note: if the Mac is asleep at fire time, launchd runs the job on next wake; if it is powered off or logged out, that morning's digest is not generated — run 'earmark run' manually`.
8. iCloud experiment hook (U3): the plist template includes a commented-out `MaterializeDatalessFiles` key with a code comment pointing at ISSUE_PLAN U3 — during wave-4 real-world validation, test whether enabling it makes `.icloud` placeholders materialize for the launchd job and record the result in U3 (enable it by default only if it demonstrably helps and has no downsides).

## Acceptance Criteria

- [ ] Golden plist snapshot for default params; XML-escape case with a path containing `& ' " <` proven byte-exact.
- [ ] Install flow exec sequence (spy): bootout → bootstrap with exact argv incl. `gui/<uid>`; flags persist to config file; plist file mode 0644.
- [ ] Entry resolution: from a simulated installed layout (temp dir with package.json + dist/cli/index.js) resolves correctly; tsx-dev context refuses with the specified message.
- [ ] Uninstall tolerant paths (not loaded / plist missing) exit 0 with accurate messages.
- [ ] Status drift matrix: moved node (nonexistent path), changed minute, missing plist, not loaded — each reported with remediation; clean install reports ok.
- [ ] `--hour 25` → exit 2 (range validation via config schema reuse).

## Validation

`vitest` with exec spies + temp dirs. Manual (dev Mac): `schedule install --hour 6 --minute 0` → `launchctl print gui/$(id -u)/dev.earmark.daily` shows the job; temporary `--minute +2` install fires a real run (with empty queue → no_articles) proving end-to-end launchd execution incl. PATH (ffmpeg found by doctor within that run's log); then `uninstall`. Attach both transcripts + `launchd.err.log` excerpt to PR (wave-4 gate evidence; also feeds U3 observation).

## Dependencies

04, 24 (`run --trigger launchd` exists).

## Non-goals

Watch-folder or interval scheduling; `pmset` wake scheduling (documented as a user option in SETUP, issue 30, not automated); multiple schedules; LaunchDaemons (user-scope only per DESIGN §13.6).

## Design References

DESIGN §11.3, §13.6, §15 (drift check consumed by doctor); ISSUE_PLAN U3.
