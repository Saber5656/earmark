# Security boundary test suite

## Summary

Add `test/security/` — a permanent, table-driven suite verifying every DESIGN §13 boundary control end-to-end: URL/scheme attacks, SSRF/redirect/private-network cases, inbox abuse (traversal/symlink/size/garbage), control-character and injection sanitization, and a static execFile/no-shell audit of the source tree.

## Context

Individual modules ship their own tests (issues 05/08/10/12), but security regressions typically arrive through refactors that reshuffle module seams. This suite pins the *externally observable* security behavior in one place, mapped 1:1 to DESIGN §13.8's abuse-case table, so a future PR cannot silently drop a control. It is a release gate (ISSUE_PLAN §6.4).

## Scope

- `test/security/{url,ssrf,inbox,sanitize,static-audit}.test.ts`; no production code changes expected (gaps found become fixes in the owning modules within this PR, each noted in the PR description).

## Detailed Requirements

1. `url.test.ts` — capture boundary (through the real `add` path, not just the lib): `javascript:`, `data:`, `file:`, `vbscript:`, `ftp:`, protocol-relative `//evil`, credentials, 2049 chars, NUL/newline embedded, unicode homoglyph host (accepted — document), punycode host (accepted); each asserts DB row count unchanged on rejection.
2. `ssrf.test.ts` — fetch boundary exercised via the issue-10 seams (injected `LookupFn` for name resolution; loopback `node:http` servers only for redirect mechanics): direct private literals (10.x, 172.16.x, 192.168.x, 169.254.x, 100.64.x, 127.1, `0.0.0.0`, `[::1]`, `[fe80::1]`, `[::ffff:10.0.0.1]`); hostname→private via injected lookup; redirect hop to each private class (origin served on loopback with a lookup that vets the origin, policy active for hops); mixed A-records (one public one private → blocked); `allowPrivateNetworks:true` bypass works; 30-hop redirect loop stops at maxRedirects; content-length lies + streamed 20 MB → capped; header-timeout slow-loris → timeout. Each case asserts the typed error code. (Simulating "public hostname" in CI = injected lookup returning a public-looking address while the actual connection goes to the local mock via the pinned-lookup seam — the same mechanism issue 10's tests use.)
3. `inbox.test.ts` — through real `ingest`, using only filesystem-representable cases (a literal `../../evil.json` basename cannot exist on POSIX): nested directory inside inbox (ignored — no recursion), basename that *looks* like encoded traversal (`..%2f..%2fevil.json` as a literal name — processed as an ordinary basename, no path interpretation), `.hidden.json`, 300-char filename, symlink → `/etc/passwd`, symlink → directory, fifo (skip on CI if creation fails), 64 KiB+1 file, `{"v":1,"url":"file:///etc/passwd"}`, deeply nested 1 MB JSON (rejected by size first), duplicate URL double-file. Asserts: no inbox-derived name or symlink target causes reads/writes outside the inbox tree (fs-spy on opened paths with an explicit allowlist for the sandbox DB, logs, and config paths), rejected files land in `rejected/`, process never crashes.
4. `sanitize.test.ts` — end-to-end text hygiene: article fixture whose title+body contain ANSI CSI/OSC, C1, bidi overrides, zero-width, `\x07`, fake log-line injection `\nERROR fake`; assert: (a) speechify output clean (regex over all paragraphs), (b) logger file lines JSON-parse and contain no raw ESC/C1 bytes, (c) DB stored title clean, (d) `list` stdout contains no ESC byte, (e) notification body builder output clean; plus SQL-injection spot check: title `'; DROP TABLE articles;--` roundtrips intact and table survives.
5. `static-audit.test.ts` — source-tree invariants (read `src/**/*.ts`, fail with file:line):
   - forbidden: `child_process.exec(`, `execSync(`, `spawnSync(` with string command, `shell: true`, `` new Function ``, `eval(`, `runScripts`, `dangerouslyRunScripts`
   - `execFile` command-argument rule: first arg must be an identifier/config-derived variable OR an absolute-path string literal from the documented allowlist `['/usr/bin/say', '/usr/bin/osascript', '/bin/launchctl']` (extendable in the test with justification comments); template literals as commands forbidden
   - `better-sqlite3` usage: no template-literal SQL containing `${` (parameterization guard heuristic)
   - undici/global fetch: no `fetch(`/`request(` outside `src/content/fetch.ts`, provider clients (`translate/ollama.ts`, `tts/voicevox.ts`, `tts/voicevox-engine.ts`), and doctor probes (allowlist file list in the test).
6. Remaining §13.8 rows so the coverage claim holds: `lock-race.test.ts` — two concurrent `acquireRunLock` calls: exactly one wins, loser gets `LOCK_HELD`; stale-lock (dead pid) steal works. Port-squat rows: mapped to the existing wrong-shape health-check tests from issues 15/17/18 — the mapping table may reference tests living elsewhere in the repo; this issue adds any that are missing rather than duplicating passing ones.
7. Each test case comments its DESIGN §13.x / §13.8 row reference; a coverage checklist in `test/security/README.md` maps **every** §13.8 row → test id (+ file), including rows satisfied by referenced module tests (reviewer verifies completeness).
7. Suite runs in the normal `npm test` (CI) — no special flag, keeps it permanent.

## Acceptance Criteria

- [ ] Every §13.8 abuse-case row has ≥ 1 passing test mapped in `test/security/README.md`.
- [ ] All §13.3 address classes covered incl. IPv6 and mixed-record cases.
- [ ] Static audit passes on the current tree AND demonstrably fails on a planted violation (add in a scratch commit in the PR to show the failure output, then remove).
- [ ] Sanitization asserted at all five observation points (speechify/logger/DB/stdout/notification).
- [ ] Any gap discovered is fixed in the owning module within this PR, with the fix listed in the PR description.

## Validation

CI green; PR description includes the coverage checklist rendered and the planted-violation demonstration. This suite is thereafter part of the release gate (ISSUE_PLAN §6).

## Dependencies

05, 06 (`add` path), 07 (`list` stdout), 08, 10, 12, 24 (lock), 25 (notification renderer) — use module entry points where the orchestrator is unnecessary.

## Non-goals

Fuzzing infrastructure; dependency CVE scanning (CI `npm audit` from issue 01); penetration testing of VOICEVOX/Ollama themselves; DNS-rebinding beyond the pinned-lookup design (documented best-effort, DESIGN §13.3).

## Design References

DESIGN §13 (entire), esp. §13.2, §13.3, §13.8, §13.1 B3/B6/B8; issues 05/08/10/12.
