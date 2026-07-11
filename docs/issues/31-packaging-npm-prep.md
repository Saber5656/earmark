# npm packaging & publish-readiness checklist

## Summary

Make the package publishable and prove it: `npm pack` content audit, packed-tarball smoke test on a clean prefix, exact-pinning audit of native/prebuilt deps, LICENSE file (after owner sign-off), CHANGELOG, repository metadata, and the documented manual publish procedure including the repository-policy history/PII scan. **Publishing itself is not part of this issue.**

## Context

v1 ends at "ready to publish" (DESIGN §18): the owner performs `npm publish` and the repository-public switch manually after this issue's checklist passes. The repo policy additionally requires scanning commit history for personal data/secrets before any public release.

## Scope

- `package.json` final fields, `LICENSE`, `CHANGELOG.md`, `docs/RELEASING.md` (the manual procedure), CI publish-dry-run job (optional, no tokens), smoke script.

## Detailed Requirements

1. `package.json` finalization: `description` (English, matches README tagline translation), `keywords`, `repository`/`bugs`/`homepage` → `github.com/Saber5656/earmark`, `author`, `license: MIT`, `files: ["dist","README.md","LICENSE","docs/SETUP.md","docs/SHORTCUT.md"]` (SETUP/SHORTCUT ship so `npm docs`-less users get them — confirm size impact < 100 KB), `engines.node: ">=22"`, `publishConfig: { access: "public", provenance: true }` (provenance only if publishing via CI later — include commented note), bin executable bit via build (`chmod` in build script or `tsc` + explicit chmod step; verify `npx` execution works from tarball).
2. **License sign-off gate**: confirm MIT with the owner (recorded decision required — if the owner has not confirmed by implementation time, STOP and ask; do not add the LICENSE file on assumption). After sign-off: standard MIT text, copyright `2026 Saber5656`.
3. Dependency pinning audit: `better-sqlite3` and `kokoro-js` pinned exact (no `^`) with a comment-doc in `docs/RELEASING.md` on the upgrade procedure; all other deps `^`-ranged; `npm ci && npm test` from a fresh clone verified; `npm audit --omit=dev` clean at high/critical.
4. Smoke test script `scripts/smoke-pack.sh`: `npm run build && npm pack` → install the tarball into a temp prefix (`npm install -g --prefix <tmp> ./earmark-0.1.0.tgz`) → run `<tmp>/bin/earmark --version`, `earmark config path`, `earmark doctor --json` (accept exit 3; assert JSON parses) with sandbox env vars → uninstall. Wire as CI job step (macos).
5. `npm pack --dry-run` content audit checklist in `docs/RELEASING.md`: MUST contain only dist/docs listed; MUST NOT contain: `test/`, fixtures, `.github/`, `docs/issues|decisions|research`, `scripts/measure-*`, any `*.wav/*.m4a`, editor/config files. Verified by the smoke script (grep the pack file list against an allowlist).
6. `CHANGELOG.md`: Keep-a-Changelog format, `0.1.0 - Unreleased` section summarizing v1 features (one line per DESIGN §1.2 row).
7. `docs/RELEASING.md` — the manual gate procedure (owner executes):
   1. all ISSUE_PLAN §6 validation items green (links)
   2. history/PII scan: run `gitleaks detect --source .` (or equivalent) AND manual `git log -p | grep`-based checklist for personal paths/emails per repo policy; **repo is currently private — the public switch happens only after this scan passes**; any hit → follow repo policy (consider fresh-repo migration)
   3. version bump conventions (semver; 0.x caveat), `git tag v0.1.0`
   4. `npm publish` (2FA note), post-publish smoke `npm install -g earmark` on a second machine/account-less check
   5. GitHub release notes from CHANGELOG.
8. CI: add the smoke-pack job; no publish automation, no tokens stored (v1 policy: manual publish only — ADR-001 zero-secret extends to CI).

## Acceptance Criteria

- [ ] `scripts/smoke-pack.sh` passes locally and in CI: tarball installs into a clean prefix and `earmark --version` + `config path` + `doctor --json` behave; pack file list matches the allowlist exactly.
- [ ] LICENSE present **with recorded owner sign-off** (PR links the confirmation) — or the issue is blocked and says so.
- [ ] `npm audit --omit=dev --audit-level=high` clean; exact-pin audit documented.
- [ ] CHANGELOG + RELEASING complete; RELEASING includes the history-scan gate wording and the private→public ordering rule.
- [ ] Fresh-clone `npm ci && npm run build && npm test` green (CI proves).
- [ ] No `npm publish` executed; no tokens/credentials introduced anywhere.

## Validation

CI smoke-pack job link + local transcript in PR. Reviewer re-runs the pack-list audit. Owner confirms license + reads RELEASING as the final v1 sign-off artifact.

## Dependencies

28, 30 (everything shippable and documented).

## Non-goals

Actual publishing / repo-public switch (owner-manual per RELEASING); Homebrew formula (v2); CI-automated releases/provenance pipeline (v2); version 1.0 semantics.

## Design References

DESIGN §18, §13.7, §2.3; ADR-001 (zero-secret incl. CI); repository policy: pre-publication history scan (user global rules — mirrored into RELEASING.md).
