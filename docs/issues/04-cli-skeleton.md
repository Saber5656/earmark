# CLI skeleton: command wiring, logger, error handler, exit codes

## Summary

Replace the issue-01 stub with the real commander-based CLI: global flags, subcommand registration pattern, the leveled logger (stderr + daily logfile with control-character stripping), central error handling mapped to the DESIGN §12.4 exit codes, and the import-layering lint rule.

## Context

Nine subcommands land across later issues; they all need one consistent entry point, one logger, one error contract (`EarmarkError` from issue 02), and machine-friendly behavior (`--json` passthrough). DESIGN §3 fixes the surface, §14.1 the logging, §12.4 the exit codes.

## Scope

- `src/cli/index.ts` (rewritten), `src/cli/context.ts` (per-invocation context object)
- `src/core/logger.ts`
- ESLint `no-restricted-imports` layering rule (DESIGN §2.2)
- Runtime dep added: `commander@^15`
- Wire the `config` command from issue 02

## Detailed Requirements

1. Program setup: `program.name('earmark').version(<from package.json>)`; global options `--verbose` (debug level), `--config <path>`, and per-command `--json` where later issues declare it. Unknown command/option → commander error → exit code 2.
2. `CliContext` built once per invocation: `{ config, logger, configPath, json: boolean }`; command handlers receive it; construction order: parse flags → load config (issue 02) → build logger from `logging.level` (verbose overrides to debug) → prune old logfiles (retentionDays, best-effort).
3. Logger (`core/logger.ts`):
   - levels `error|warn|info|debug`; API `logger.info(msg, ctx?)` with `ctx` as flat JSON-safe object
   - sinks: stderr (human format `HH:MM:SS LEVEL msg key=value`, colors only when TTY) and daily file `<stateDir>/logs/earmark-YYYY-MM-DD.log` as JSON lines `{ts, level, msg, ...ctx}`
   - **sanitization**: strip C0 (except `\n` in file sink it becomes `\\n`), C1, ANSI CSI/OSC sequences, bidi controls from `msg` and string ctx values before writing (DESIGN §13.1 B8)
   - file sink failures (unwritable dir) degrade to stderr warning once, never crash
   - `pruneLogs(retentionDays)` deletes `earmark-*.log` older than N days by filename date.
4. Central error handler wrapping every command action:
   - `EarmarkError` → `logger.error` + stderr line `error (<CODE>): <message>` → `process.exitCode = err.exitCode`
   - zod/`CONFIG_INVALID` → exit 3; unexpected `Error` → stack at debug, one-line message, exit 1
   - `--json` mode: errors also emit `{"error": {"code": ..., "message": ...}}` to stdout as the only stdout output.
5. Exit codes exactly DESIGN §12.4: 0 success/no-op; 1 unexpected/environmental; 2 usage; 3 config/doctor validation; 4 lock held. Export `EXIT_CODES` constant map for reuse.
6. Command registration pattern: each `src/cli/commands/<name>.ts` exports `register<Name>(program, getCtx)`; `index.ts` imports and registers all (commented placeholders for commands from later issues, added as they land).
7. Layering lint (DESIGN §2.2): ESLint `no-restricted-imports` (or `import/no-restricted-paths` via `eslint-plugin-import`) configured so `src/core/**` cannot import from `src/cli/**` or feature dirs; feature dirs (`capture|content|translate|script|tts|audio|pipeline|schedule|doctor`) cannot import `src/cli/**`; feature→feature imports only from the `types.ts` interface files and `pipeline/**`. A violating import must fail `npm run lint` (prove with a temporary test).
8. Startup must not touch the DB (only commands that need it open it) — keeps `earmark config`/`--version` working when better-sqlite3 native binding is broken; that failure path shows a remediation hint (reinstall).

## Acceptance Criteria

- [ ] `earmark --version`, `earmark --help`, `earmark config path` work end-to-end via `npm run dev --`.
- [ ] `earmark nonexistent` → exit 2 with commander usage error; `EARMARK_CONFIG_DIR` pointing at a corrupt config → exit 3 with zod detail.
- [ ] Logfile contains JSON lines with stripped ANSI/bidi when a crafted `ctx` value includes `[31m` and `‮` (unit test).
- [ ] TTY vs non-TTY stderr formatting verified (force via env `FORCE_COLOR`/`NO_COLOR` conventions: honor `NO_COLOR`).
- [ ] Layering rule fails lint on a deliberate `core → cli` import (test added then reverted, demonstrated in PR).
- [ ] Exit-code constants used, no bare `process.exit(n)` literals in command files.

## Validation

Unit tests: logger sanitization table, prune-by-date logic (fixture filenames), error-handler mapping matrix. Manual transcript in PR: the five exit-code scenarios with `echo $?` after each.

## Dependencies

01, 02.

## Non-goals

Implementations of add/list/ingest/run/schedule/doctor/log (issues 06–27); notifications (25); localization of CLI messages (English CLI output is fine for v1; digest narration i18n is separate — DESIGN §9.2).

## Design References

DESIGN §3 (CLI surface), §12.4 (exit codes), §14.1 (logging), §2.2 (layering), §13.1 B8 (log injection).
