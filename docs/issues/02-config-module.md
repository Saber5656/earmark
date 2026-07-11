# Config module: schema, defaults, atomic persistence, `earmark config`

## Summary

Implement `src/core/paths.ts` and `src/core/config.ts`: the zod schema for the full configuration in DESIGN §4.1, default generation, atomic load/save, env overrides, and the `earmark config get|set|path` command **as an exported command factory** (CLI wiring happens in issue 04).

## Context

Every module reads typed config (paths, network limits, provider settings). Config is a single JSON file validated on every load; typos must fail loudly (DESIGN §4). This issue also owns filesystem path conventions (XDG-style dirs, iCloud defaults — DESIGN §6, ADR-002) and the project error class `EarmarkError`.

## Scope

- `src/core/paths.ts`: app dir resolution + `~` expansion
- `src/core/errors.ts`: `EarmarkError { code: string; exitCode: number; cause? }`
- `src/core/config.ts`: zod schema, defaults, load/save, dotted-path get/set
- `src/cli/commands/config.ts`: exported command factory + action functions, unit-tested directly (no runnable CLI yet — issue 04 registers it)
- Runtime dep added: `zod@^4`

## Detailed Requirements

1. `paths.ts`:
   - `expandTilde(p)`: replaces leading `~/` with `os.homedir()`; absolute paths pass through unchanged.
   - Path-field rule for config values under `paths.*` EXCEPT `ffmpegPath`/`ffprobePath`: must be absolute or start with `~/`; anything else fails validation. `ffmpegPath`/`ffprobePath` may be either a bare command name (resolved via `PATH` at spawn time) or an absolute path.
   - Resolved dir accessors honoring env overrides: `configDir()` = `$EARMARK_CONFIG_DIR` or `~/.config/earmark`; data/state/cache dirs come from config `paths.*` (defaults per DESIGN §4.1).
   - `resolvedInboxDir(cfg)` / `resolvedOutputDir(cfg)`: explicit config value if set (string), else `<icloudRoot>/inbox` and `<icloudRoot>/digests`.
   - `ensureDir(p)`: `mkdir -p`; used by callers, never automatically at import time.
2. Zod schema: exactly the keys/defaults/types/ranges of DESIGN §4.1. Additional constraints:
   - `.strict()` on every object level (unknown keys → error listing the offending path)
   - `outputLanguage`: `z.enum(['ja','en'])`; `digest.maxArticles`: int 1–50; `schedule.hour` 0–23, `minute` 0–59; `translation.provider`: `z.enum(['ollama'])`; `tts.ja.provider`: `z.enum(['voicevox'])`; `tts.en.provider`: `z.enum(['kokoro','say'])`; `tts.ja.voicevox.speedScale`: 0.5–2.0; ms/byte fields positive ints
   - provider base URLs (`translation.ollama.baseUrl`, `tts.ja.voicevox.baseUrl`): must parse with `new URL` and have scheme `http:` or `https:`, else config validation error
   - `network.userAgent` default is the literal template string `earmark/{version} (+https://github.com/Saber5656/earmark)`; the `{version}` placeholder is substituted by consumers at use time (issue 10), NOT at config creation — the stored file stays stable across upgrades
   - Export `type EarmarkConfig = z.infer<...>` and `DEFAULT_CONFIG()` (returns fresh defaults).
3. `loadConfig(opts?: {configPath?: string})`:
   - path = explicit `--config` value, else `<configDir()>/config.json`. Explicit `configPath` contract: absolute or `~/`-prefixed only; otherwise throw `EarmarkError` code `CONFIG_INVALID` (exit code 3). Parent dir of an explicit path is created only when creating a missing config file.
   - file missing → create parent dir + write defaults (atomic) + return defaults, log info "created default config"
   - parse/validation errors → `EarmarkError CONFIG_INVALID` (exit 3), message includes zod issue paths.
4. `saveConfig(cfg, path)`: validate → `JSON.stringify(cfg, null, 2) + "\n"` → write to `<dir>/.config.json.<pid>.<random6>.tmp` in the same directory → `fs.renameSync` over target (atomic on same volume); temp file unlinked on failure (best effort).
5. Dotted-path access: `getByPath(cfg, 'tts.ja.voicevox.speaker')` and `setByPath(cfg, key, rawValue)`:
   - `rawValue` parsing: try `JSON.parse`; on failure treat as string (so `set digest.maxArticles 15` yields number, `set outputLanguage en` yields string, `set paths.inboxDir null` yields null)
   - after set, whole config re-validated; invalid → error, file untouched.
6. `config` command actions (exported functions `configPathAction/configGetAction/configSetAction` plus a commander `Command` factory consumed by issue 04):
   - `config path` → prints resolved config file path
   - `config get` → pretty-prints entire effective config (JSON); `config get <key>` → prints value as JSON
   - `config set <key> <value>` → validates + saves atomically, prints `ok: <key> = <value-json>`; unknown key or invalid value → `CONFIG_INVALID` (exit 3 once wired).
7. Loopback helpers: `isLoopbackUrl(urlString)` → true for hosts in `127.0.0.0/8`, `[::1]`, `localhost` (case-insensitive). `validateProviderUrls(cfg): string[]` returns one warning string per non-loopback provider base URL; v1 checks exactly two fields: `translation.ollama.baseUrl` and `tts.ja.voicevox.baseUrl`. Warning emission happens at provider construction (issues 15/17); this module only supplies the helper.

## Acceptance Criteria

- [ ] Fresh machine simulation: `loadConfig()` with empty temp `$EARMARK_CONFIG_DIR` creates `config.json` matching `DEFAULT_CONFIG()` byte-for-byte (2-space indent + trailing newline), including the literal `{version}` placeholder in `userAgent`.
- [ ] A unit test compares `DEFAULT_CONFIG()` against a checked-in expected object covering **every** key of DESIGN §4.1 (drift in either direction fails), plus range tests for each constrained numeric/enum field (min, max, and one out-of-range case each).
- [ ] Unknown key in file (e.g. `"outputLangage"`) → `CONFIG_INVALID` naming the bad path; malformed provider base URL (`not-a-url`, `ftp://x`) → `CONFIG_INVALID`.
- [ ] `set`/`get` round-trip works for string/number/boolean/null cases listed above; invalid set leaves the file byte-identical.
- [ ] Atomicity: save writes a same-directory dot-tmp file then renames (fs spies); no partial config observable.
- [ ] Path rules: `paths.dataDir` set to `relative/dir` → `CONFIG_INVALID`; `ffmpegPath: "ffmpeg"` accepted; `--config ./rel.json` → `CONFIG_INVALID`.
- [ ] `validateProviderUrls` table: `http://192.168.1.5:50021` → 1 warning; `http://127.0.0.1:50021`, `http://[::1]:11434`, `http://LOCALHOST:11434`, `https://127.0.0.1:1` → 0 warnings.

## Validation

`vitest` unit suite with a temp dir per test (`fs.mkdtemp`): default creation, strict rejection, dotted set/get matrix, atomicity, tilde expansion, inbox/output dir resolution with and without explicit overrides, loopback helper table, command action functions invoked directly with captured stdout. (CLI-level `earmark config …` smoke happens in issue 04's validation.)

## Dependencies

01.

## Non-goals

CLI registration/global flags (issue 04); env-var overrides of individual keys; config migration logic (schema `version` field exists; first migration ships when needed); warning emission at provider construction (issues 15/17).

## Design References

DESIGN §4 (schema), §6 (layout), §13.6 (loopback posture), §3 (`config` command), §12.4 (exit codes); ADR-002.
