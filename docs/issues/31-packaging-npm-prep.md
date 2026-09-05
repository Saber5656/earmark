# npm packaging & publish-readiness checklist

## Summary

Make the package publishable and prove it: `npm pack` content audit against an exact allowlist, packed-tarball smoke test on a clean prefix in a sandboxed environment, exact-pinning audit of native/prebuilt deps (incl. transitive `onnxruntime-node`), LICENSE file (only after owner sign-off), CHANGELOG, repository metadata, and the documented manual publish procedure including the repository-policy history/PII scan. **Publishing itself is not part of this issue.**

## Context

v1 ends at "ready to publish" (DESIGN §18): the owner performs `npm publish` manually after this issue's checklist passes. The GitHub repository is already public; the release-hygiene gates here protect the npm artifact and the tagged release. Repo policy requires a commit-history personal-data/secret scan before publication artifacts go out.

## Scope

- `package.json` final fields, `LICENSE`, `CHANGELOG.md`, `docs/RELEASING.md`, CI smoke-pack job, `scripts/smoke-pack.sh`.

## Detailed Requirements

1. `package.json` finalization (exact values — do not invent):
   - `description`: `Turn read-later articles into a daily local TTS podcast digest (macOS)`
   - `keywords`: `["tts","podcast","read-later","text-to-speech","voicevox","kokoro","cli","macos"]`
   - `repository`: `{"type":"git","url":"git+https://github.com/Saber5656/earmark.git"}`; `bugs`: `https://github.com/Saber5656/earmark/issues`; `homepage`: `https://github.com/Saber5656/earmark#readme`
   - `author`: `Saber5656`
   - `license`: `MIT`; `engines.node: ">=22"`; `files: ["dist","README.md","LICENSE"]` (DESIGN §18 — docs stay on GitHub)
   - `publishConfig`: `{"access":"public"}` only — **no `provenance`** (requires CI/OIDC publishing; v1 is manual; future CI provenance is a RELEASING.md note)
   - bin executable: build step ensures `dist/cli/index.js` keeps its shebang and the pack/install path yields an executable `earmark` (verified by the smoke script).
2. **License** (owner-confirmed 2026-07-11 during the v1 requirements session; recorded in the design PR): MIT, copyright line `Copyright (c) 2026 Saber5656`. Add the standard MIT `LICENSE` file accordingly.
3. Dependency pinning audit:
   - `better-sqlite3` and `kokoro-js` pinned exact (no `^`)
   - transitive `onnxruntime-node` (via kokoro-js) pinned via `package.json` `overrides` to the exact version kokoro-js@1.2.1 resolves to; evidence: `npm ls onnxruntime-node better-sqlite3 kokoro-js` output in `docs/RELEASING.md`
   - all other deps `^`-ranged; fresh-clone `npm ci && npm run build && npm test` green (CI proves); `npm audit --omit=dev --audit-level=high` clean.
4. Lifecycle-script gate: `package.json` must contain no `preinstall`/`install`/`postinstall`/`prepare`-with-side-effects scripts of our own (`prepare` for build-on-git-install is also disallowed in v1 — packing is explicit); an automated check in the smoke script greps for them. Any new/modified GitHub Actions in this PR pinned to full commit SHAs (DESIGN §13.7).
5. `scripts/smoke-pack.sh` (runs locally and as a CI job on macos):
   - `npm run build && npm pack` → capture tarball
   - allowlist audit: `tar -tzf` file list must equal exactly: `package/package.json`, `package/README.md`, `package/LICENSE`, and `package/dist/**` (nothing else — npm always includes package.json/README/LICENSE regardless of `files`; the assertion is an exact-set comparison after globbing dist)
   - sandboxed install: `npm install -g --prefix "$TMP/prefix" ./earmark-*.tgz`; then run with sandbox env: `EARMARK_CONFIG_DIR="$TMP/cfg"` and a pre-written config setting `paths.dataDir/stateDir/cacheDir/icloudRoot/inboxDir/outputDir` all under `$TMP` — commands: `earmark --version` (exact match), `earmark config path`, `earmark doctor --json` (exit 3 acceptable; assert stdout parses as JSON and no file outside `$TMP`/the prefix was created — spot-check `~/.config/earmark` untouched)
   - uninstall + cleanup; lifecycle-script grep (req 4).
6. `CHANGELOG.md`: Keep-a-Changelog format, `## [0.1.0] - Unreleased` summarizing v1 features (one line per DESIGN §1.2 row).
7. `docs/RELEASING.md` — the manual gate procedure (owner executes):
   1. all ISSUE_PLAN §6 validation items green (links)
   2. history/PII scan of the repository: `gitleaks detect --source .` (or equivalent) AND a manual checklist (`git log -p` grep for home paths, emails, tokens) per repo policy; any hit → follow policy (fresh-repo migration consideration) **before tagging or publishing**
   3. version conventions (semver, 0.x caveat), `git tag v0.1.0`
   4. `npm publish` (2FA note) + post-publish smoke (`npm install -g earmark` on a clean prefix)
   5. GitHub release notes from CHANGELOG
   6. future work note: CI-based publishing with `--provenance` (v2; requires OIDC setup — explains why v1 omits it).

## Acceptance Criteria

- [ ] `scripts/smoke-pack.sh` passes locally and in CI: exact-set tarball allowlist; sandboxed global install runs the three commands; nothing written outside the sandbox; lifecycle-script grep clean.
- [ ] LICENSE present: standard MIT text, `Copyright (c) 2026 Saber5656` (owner-confirmed 2026-07-11; the design PR records the decision).
- [ ] `npm ls` pinning evidence for `better-sqlite3`, `kokoro-js`, `onnxruntime-node` (overrides effective) recorded in RELEASING.md; `npm audit --omit=dev --audit-level=high` clean.
- [ ] `package.json` fields byte-match req 1 (test or reviewer diff against the literal values above).
- [ ] CHANGELOG + RELEASING complete; RELEASING includes the history-scan gate and the provenance deferral note.
- [ ] No `npm publish` executed; no tokens/credentials introduced anywhere (CI has no registry secrets).

## Validation

CI smoke-pack job link + local transcript in PR. Reviewer re-runs the tarball allowlist audit. Owner confirms license/author values and accepts RELEASING.md as the final v1 sign-off artifact.

## Dependencies

28, 30 (everything shippable and documented).

## Non-goals

Actual publishing (owner-manual per RELEASING); Homebrew formula (v2); CI-automated releases/provenance (v2); shipping SETUP/SHORTCUT docs inside the npm tarball (GitHub is their home — DESIGN §18); version 1.0 semantics.

## Design References

DESIGN §18, §13.7, §2.3; ADR-001 (zero-secret incl. CI); repository policy: pre-publication history scan (mirrored into RELEASING.md).
