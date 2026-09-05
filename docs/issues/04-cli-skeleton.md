# CLI skeleton: command wiring, logger, error handler, exit codes

## Summary

Replace the issue-01 stub with the real commander-based CLI: global flags, subcommand registration pattern, the leveled logger (stderr + daily logfile with control-character stripping), central error handling mapped to the DESIGN §12.4 exit codes, and the import-layering lint rule.

## Context

Nine subcommands land across later issues; they all need one consistent entry point, one logger, one error contract (`EarmarkError` from issue 02), and machine-friendly behavior (`--json` passthrough). DESIGN §3 fixes the surface, §14.1 the logging, §12.4 the exit codes.

## Scope

- `src/cli/index.ts` (rewritten), `src/cli/context.ts` (per-invocation context object)
- `src/core/logger.ts`, `src/core/text.ts` (canonical sanitizer)
- ESLint `no-restricted-imports` layering rule (DESIGN §2.2)
- Runtime dep added: `commander@^15`
- Wire the `config` command factory from issue 02

## Detailed Requirements

1. Program setup: `program.name('earmark').version(<from package.json>)`; global options `--verbose` (debug level), `--config <path>`, and per-command `--json` where later issues declare it. Unknown command/option → commander error → exit code 2.
2. `CliContext` built once per invocation: `{ config, logger, configPath, json: boolean }`; command handlers receive it; construction order: parse flags → load config (issue 02) → build logger from `logging.level` (verbose overrides to debug). Commands may declare `lazyConfig: true` in their registration (used by `doctor`, issue 27) — for those, config load errors are captured into the context (`configError`) instead of aborting, so the command can report them itself. Log pruning is NOT done at startup; `pruneLogs` is exported and called by `earmark run` (issue 24) per DESIGN §14.1.
3. Canonical sanitizer (`core/text.ts`): `stripControl(s: string): string` removes, in order: ANSI escape sequences (CSI `\x1b[...cmd` and OSC `\x1b]...(\x07|\x1b\\)`), remaining C0 controls except `\n`, C1 (U+0080–U+009F), zero-width (U+200B–U+200D, U+FEFF), and bidi controls (U+202A–U+202E, U+2066–U+2069). This is the single project-wide sanitizer referenced by issues 06/07/08/11/21/22/25 and DESIGN §8.3 rule 11.
4. Logger (`core/logger.ts`):
   - levels `error|warn|info|debug`; API `logger.info(msg, ctx?)` with `ctx` as flat JSON-safe object
   - sinks: stderr (human format `HH:MM:SS LEVEL msg key=value`, colors only when TTY and `NO_COLOR` unset) and daily file `<stateDir>/logs/earmark-YYYY-MM-DD.log` as JSON lines with **nested** ctx: `{"ts": "...", "level": "...", "msg": "...", "ctx": {...}}` (DESIGN §14.1)
   - **sanitization**: `stripControl` applied to `msg` and every string ctx value before writing; `\n` inside values is escaped by JSON serialization in the file sink and replaced with `␤` in the stderr sink
   - file sink failures (unwritable dir) degrade to a one-time stderr warning, never crash
   - `pruneLogs(retentionDays)` deletes `earmark-*.log` older than N days by filename date.
5. Central error handler wrapping every command action:
   - `EarmarkError` → `logger.error` + stderr line `error (<CODE>): <message>` → `process.exitCode = err.exitCode`
   - zod/`CONFIG_INVALID` → exit 3; unexpected `Error` → stack at debug, one-line message, exit 1
   - `--json` mode: errors also emit `{"error": {"code": ..., "message": ...}}` to stdout as the only stdout output
   - bootstrap failures **before** the logger exists (config load/parse in step 2 for non-lazy commands): write the sanitized one-line error directly to stderr (plus the JSON error object to stdout when `--json` was parsed), exit 3 — no logger involved.
6. Exit codes exactly DESIGN §12.4: 0 success/no-op; 1 unexpected/environmental; 2 usage; 3 config/doctor validation; 4 lock held. Export `EXIT_CODES` constant map for reuse. CLI-only codes such as `NOT_FOUND` (issue 07) and `USAGE` live beside pipeline codes in `core/errors.ts` with a comment separating them from the DESIGN §12.2 article codes.
7. Command registration pattern: each `src/cli/commands/<name>.ts` exports `register<Name>(program, getCtx)`; `index.ts` imports and registers all (commented placeholders for commands from later issues, added as they land). `--json` is a per-command option declared by commands that support it (DESIGN §3), not a global flag.
8. Layering lint (DESIGN §2.2): ESLint `no-restricted-imports` (or `import/no-restricted-paths` via `eslint-plugin-import`) configured so `src/core/**` cannot import from `src/cli/**` or feature dirs; feature dirs (`capture|content|translate|script|tts|audio|pipeline|schedule|doctor`) cannot import `src/cli/**`; feature→feature imports only from the `types.ts` interface files and `pipeline/**`. Persistent proof: a vitest test invokes the ESLint Node API on a temp file containing a violating import (written under a temp dir mapped into the lint config for the test) and asserts the rule errors — the negative case is exercised on every `npm test`, not via a one-off reverted commit.
9. Startup must not import or open the DB (only commands that need it do) — keeps `earmark config`/`--version` working when the better-sqlite3 native binding is broken; the remediation hint for a broken binding is owned by the first DB-opening command path (issue 06).

## Acceptance Criteria

- [ ] `earmark --version`, `earmark --help`, `earmark config path` work end-to-end via `npm run dev --`.
- [ ] `earmark nonexistent` → exit 2 with commander usage error; `EARMARK_CONFIG_DIR` pointing at a corrupt config → exit 3 with zod detail.
- [ ] Logfile lines JSON-parse with the nested `ctx` shape and contain no ESC/C1/bidi bytes when a crafted `ctx` value includes an ANSI CSI sequence and U+202E (unit test on the file sink).
- [ ] `stripControl` table-driven test: ANSI CSI, OSC, C0, C1, zero-width, bidi — each removed; plain ja/en text with emoji untouched.
- [ ] TTY behavior: color codes present with mocked `process.stderr.isTTY = true`, absent when `isTTY = false`; separately, `NO_COLOR=1` disables color even on a TTY.
- [ ] Layering rule: the persistent ESLint-API negative test passes (violating fixture errors, compliant fixture passes) on every `npm test`.
- [ ] Bootstrap failure path: corrupt config + non-lazy command → sanitized stderr line, exit 3, no logfile created.
- [ ] Exit-code constants used, no bare `process.exit(n)` literals in command files.

## Validation

Unit tests: logger sanitization table, prune-by-date logic (fixture filenames), error-handler mapping matrix. Manual transcript in PR: the five exit-code scenarios with `echo $?` after each.

## Dependencies

01, 02.

## Non-goals

Implementations of add/list/ingest/run/schedule/doctor/log (issues 06–27); notifications (25); localization of CLI messages (English CLI output is fine for v1; digest narration i18n is separate — DESIGN §9.2).

## Design References

DESIGN §3 (CLI surface), §12.4 (exit codes), §14.1 (logging), §2.2 (layering), §13.1 B8 (log injection).
