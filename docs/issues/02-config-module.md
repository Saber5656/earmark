# Config module: schema, defaults, atomic persistence, `earmark config`

## Summary

Implement `src/core/paths.ts` and `src/core/config.ts`: the zod schema for the full configuration in DESIGN §4.1, default generation, atomic load/save, env overrides, and the `earmark config get|set|path` CLI subcommand.

## Context

Every module reads typed config (paths, network limits, provider settings). Config is a single JSON file validated on every load; typos must fail loudly (DESIGN §4). This issue also owns filesystem path conventions (XDG-style dirs, iCloud defaults — DESIGN §6, ADR-002).

## Scope

- `src/core/paths.ts`: app dir resolution + `~` expansion
- `src/core/config.ts`: zod schema, defaults, load/save, dotted-path get/set
- `src/cli/commands/config.ts`: subcommand (wired into the CLI skeleton when issue 04 lands; until then export the command object and cover by unit tests)
- Runtime dep added: `zod@^4`

## Detailed Requirements

1. `paths.ts`:
   - `expandTilde(p)`: replaces leading `~/` with `os.homedir()`; no other expansion; absolute paths pass through; relative paths (no `~`, not absolute) are rejected with a validation error when they appear in config path fields.
   - Resolved dir accessors honoring env overrides: `configDir()` = `$EARMARK_CONFIG_DIR` or `~/.config/earmark`; data/state/cache dirs come from config `paths.*` (defaults per DESIGN §4.1).
   - `resolvedInboxDir(cfg)` / `resolvedOutputDir(cfg)`: explicit config value if set (string), else `<icloudRoot>/inbox` and `<icloudRoot>/digests`.
   - `ensureDir(p)`: `mkdir -p` with mode default; used by callers, never automatically at import time.
2. Zod schema: exactly the keys/defaults/types/ranges of DESIGN §4.1. Additional constraints:
   - `.strict()` on every object level (unknown keys → error listing the offending path)
   - `outputLanguage`: `z.enum(['ja','en'])`; `digest.maxArticles`: int 1–50; `schedule.hour` 0–23, `minute` 0–59; `translation.provider`: `z.enum(['ollama'])`; `tts.ja.provider`: `z.enum(['voicevox'])`; `tts.en.provider`: `z.enum(['kokoro','say'])`; `tts.ja.voicevox.speedScale`: 0.5–2.0; ms/byte fields positive ints.
   - `userAgent` default contains the real package version (interpolated at load, not stored).
   - Export `type EarmarkConfig = z.infer<...>` and `DEFAULT_CONFIG` (a function returning fresh defaults).
3. `loadConfig(opts?: {configPath?: string})`:
   - path = explicit `--config` value, else `<configDir()>/config.json`
   - file missing → create parent dir + write defaults (atomic) + return defaults, log info "created default config"
   - parse errors / validation errors → throw `EarmarkError` with code `CONFIG_INVALID`, exit code 3, message including zod issue paths (issue 04 maps this; until then the error class lives here in `core/errors.ts` — create it: `EarmarkError { code: string; exitCode: number; cause? }`).
4. `saveConfig(cfg, path)`: validate → `JSON.stringify(cfg, null, 2) + "\n"` → write to `path + ".tmp"` in same dir → `fs.renameSync` over target (atomic on same volume).
5. Dotted-path access: `getByPath(cfg, 'tts.ja.voicevox.speaker')` and `setByPath(cfg, key, rawValue)`:
   - `rawValue` parsing: try `JSON.parse`; on failure treat as string (so `earmark config set digest.maxArticles 15` yields number, `set outputLanguage en` yields string, `set paths.inboxDir null` yields null)
   - after set, whole config re-validated; invalid → error, file untouched.
6. `earmark config` subcommand behavior:
   - `config path` → prints resolved config file path
   - `config get` → pretty-prints entire effective config (JSON); `config get <key>` → prints value as JSON
   - `config set <key> <value>` → validates + saves atomically, prints `ok: <key> = <value-json>`; unknown key or invalid value → exit 3 with the zod message
7. Loopback helper for later issues: `isLoopbackUrl(urlString)` → true for hosts `127.0.0.0/8`, `::1`, `localhost`. Non-loopback provider base URLs produce a `logger.warn` at load-time consumer sites (the warning emission itself happens where providers are constructed — issues 15/17; here only the helper + a `validateProviderUrls(cfg): string[]` returning warning strings, unit-tested).

## Acceptance Criteria

- [ ] Fresh machine simulation: `loadConfig()` with empty temp `$EARMARK_CONFIG_DIR` creates `config.json` matching `DEFAULT_CONFIG()` byte-for-byte (2-space indent + trailing newline).
- [ ] Unknown key in file (e.g. `"outputLangage"`) → `EarmarkError CONFIG_INVALID` naming the bad path.
- [ ] `set`/`get` round-trip works for string/number/boolean/null cases listed above; invalid set leaves the file byte-identical.
- [ ] Save is atomic: no partially-written config observable (tested via tmp-file naming convention and rename call; inspect with fs spies).
- [ ] All defaults and ranges match DESIGN §4.1 exactly (reviewer diff-checks the table).
- [ ] `validateProviderUrls` flags `http://192.168.1.5:50021` and passes `http://127.0.0.1:50021`.

## Validation

`vitest` unit suite with a temp dir per test (`fs.mkdtemp`): default creation, strict rejection, dotted set/get matrix, atomicity, tilde expansion, inbox/output dir resolution with and without explicit overrides, loopback helper table. Manual: `EARMARK_CONFIG_DIR=$(mktemp -d) npm run dev -- config path && ... config set digest.maxArticles 5 && ... config get digest.maxArticles`.

## Dependencies

01.

## Non-goals

Reading config from project-local files or env-var overrides of individual keys; config migration logic (schema `version` field exists; first migration ships when needed); CLI wiring polish (issue 04); warning emission at provider construction (issues 15/17).

## Design References

DESIGN §4 (schema), §6 (layout), §13.6 (loopback posture), §3 (`config` command); ADR-002.
