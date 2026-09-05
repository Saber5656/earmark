# Project scaffold: TypeScript package, lint/test toolchain, CI

## Summary

Create the npm package skeleton for `earmark`: strict TypeScript (ESM), build/lint/format/test toolchain, a minimal CLI entry stub, and a GitHub Actions CI workflow on macOS. Establishes the module-layout and dependency rules every later issue relies on.

## Context

earmark is a fully local macOS CLI (DESIGN §1). All 30 subsequent issues add modules into the layout defined in DESIGN §2.2. This issue creates that skeleton with working quality gates so every later PR runs typecheck/lint/tests in CI from day one.

## Scope

- `package.json`, `tsconfig.json`, ESLint (flat config) + Prettier, Vitest config
- `src/cli/index.ts` stub (version output only), directory skeleton with `.gitkeep`-free empty dirs omitted (create dirs only when first file lands; only `src/cli/` exists now)
- `.github/workflows/ci.yml`, `.gitignore`, `.editorconfig`
- No feature code, no runtime dependencies beyond what the stub needs

## Detailed Requirements

1. `package.json`:
   - `name: "earmark"`, `version: "0.1.0"`, `private: false`, `type: "module"`, `license: "MIT"` (LICENSE file itself is added in issue 31 after owner sign-off)
   - `bin: { "earmark": "dist/cli/index.js" }`, `main` omitted, `files: ["dist", "README.md", "LICENSE"]`
   - `engines: { "node": ">=22" }`
   - scripts: `build` (`tsc -p tsconfig.json`), `typecheck` (`tsc --noEmit`), `lint` (`eslint .`), `format` (`prettier --write .`), `format:check` (`prettier --check .`), `test` (`vitest run`), `dev` (`tsx src/cli/index.ts`)
   - devDependencies: `typescript@^5`, `tsx`, `vitest@^3`, `eslint@^9` + `typescript-eslint`, `prettier@^3`, `@types/node@^22`
   - runtime dependencies: none yet (added by later issues as specified there)
2. `tsconfig.json`: `strict: true`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `target: "ES2023"`, `outDir: "dist"`, `rootDir: "src"`, `declaration: false`, `sourceMap: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`, include `src`.
3. `src/cli/index.ts`: shebang `#!/usr/bin/env node`; prints `earmark <version>` (read from `package.json` via `createRequire` or `fs`) for `--version`/`-V`; any other invocation prints one line `earmark: not yet implemented (see docs/ISSUE_PLAN.md)` and exits 0. No commander yet (issue 04).
4. ESLint flat config: typescript-eslint recommended-type-checked base; rules enforced as errors: `no-restricted-imports` placeholder (populated in issue 04 with the layering rule), `@typescript-eslint/no-floating-promises`, `no-console` **off** for `src/cli/**` and on elsewhere (feature modules must use the logger from issue 04; until then nothing else exists).
5. CI workflow `.github/workflows/ci.yml`:
   - trigger: `push` to any branch, `pull_request`
   - single job on `macos-14`: checkout → setup-node (Node 22, cache npm) → `npm ci` → `npm run typecheck` → `npm run lint` → `npm run format:check` → `npm test` → `npm audit --omit=dev --audit-level=high`
   - All third-party actions pinned to a full commit SHA (look up the current SHA for the major version at implementation time; do not use floating tags) — DESIGN §13.7.
6. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `.DS_Store`, `*.tsbuildinfo`, `.eslintcache`.
7. One placeholder test `test/smoke.test.ts` asserting the version string in `package.json` matches `/^\d+\.\d+\.\d+$/` so `vitest run` passes with a real test.
8. `npm audit` in CI must pass at setup time; commit `package-lock.json`.

## Acceptance Criteria

- [ ] `npm ci && npm run build` succeeds on macOS with Node ≥ 22; `node dist/cli/index.js --version` prints `earmark 0.1.0`.
- [ ] `npm run typecheck`, `lint`, `format:check`, `test` all pass locally.
- [ ] CI workflow runs all gates on `macos-14` and passes on the PR; all actions SHA-pinned.
- [ ] `package.json` fields exactly as specified (bin/files/engines/type).
- [ ] No runtime dependencies introduced.

## Validation

Run locally: `npm ci`, `npm run build`, `node dist/cli/index.js --version`, `npm test`. Push branch; attach green CI run link to the PR. `npx --yes .` smoke from a packed tarball is deferred to issue 31.

## Dependencies

None (first issue).

## Non-goals

Commander wiring, logger, config (issues 02/04); LICENSE file (issue 31); any feature module; README changes (issue 30).

## Design References

DESIGN §2.2 (module layout), §2.3 (dependency policy), §13.7 (supply chain), §16 (CI); ADR-002.
