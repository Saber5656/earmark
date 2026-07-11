# Setup & usage documentation (README, SETUP)

## Summary

Write the user-facing documentation: a rewritten Japanese-first `README.md` (concept, quickstart, command reference) and `docs/SETUP.md` (English, with Japanese quick-block) covering prerequisites, installation, first digest, scheduling, troubleshooting, and uninstall. Execute and record the real-world acceptance checklist (ISSUE_PLAN §6.5).

## Context

earmark's success depends on morning-one reliability for a non-expert setting it up from scratch: three external tools (ffmpeg, VOICEVOX, Ollama+model), an iOS Shortcut, and a launchd agent. Doctor (issue 27) automates diagnosis; the docs provide the narrative around it. README stays Japanese-first per repository language policy (DESIGN preamble); SETUP is English for OSS reach.

## Scope

- `README.md` (rewrite), `docs/SETUP.md` (new); conditional `docs/ISSUE_PLAN.md` edits when the acceptance checklist produces U3/U5 observations. Links to `docs/SHORTCUT.md` (issue 09). No code changes except fixing doc-revealed message typos (allowed, listed in PR).

## Detailed Requirements

1. `README.md` (Japanese, concise — details live in SETUP):
   - tagline (keep existing line) + 3-sentence concept + one architecture diagram (ASCII, simplified from DESIGN §2.1)
   - 特徴 (features) bullet list incl. 完全ローカル・シークレットゼロ (ADR-001 property, stated as a user promise), 毎朝1本のチャプター付きダイジェスト, ja/en 出力
   - クイックスタート: install → `earmark doctor` → follow remediations → `earmark add <url>` → `earmark run` → listen; then `earmark schedule install`
   - コマンド一覧 table (from DESIGN §3, one-line each)
   - 制約 (v1 constraints, honest): paywall/JS pages fail visibly; sleeping Mac → digest on wake, powered-off/logged-out Mac → that morning's digest is skipped (run manually); translation quality depends on local model
   - English one-paragraph summary + link to SETUP.md. (License line is added by issue 31 after the owner sign-off — not this issue.)
2. `docs/SETUP.md` (English; ~equivalent Japanese quick-block at top):
   - prerequisites matrix with install commands: Homebrew ffmpeg; VOICEVOX (GUI app download URL + the headless engine alternative, matching issue 18's discovery paths); Ollama (`brew install ollama`, `ollama pull gemma3:12b`, model alternatives table from research/local-translation-selection.md incl. RAM guidance); Node ≥ 22
   - install earmark: from checkout (exact sequence: `git clone … && cd earmark && npm ci && npm run build && npm install -g .`) and from the registry (`npm install -g earmark`, clearly labeled "after the first release" and exempt from copy-paste verification until published)
   - first-run walkthrough with expected doctor output evolution (before/after each install — use the real outputs captured in issue 27's validation)
   - configuration guide: full key reference table (mirrors DESIGN §4.1 — mark DESIGN as source of truth and keep the table generated-by-hand-in-sync note), common recipes: change voice (`listSpeakers` via doctor output), speed 1.2, output language en (kokoro download note + say fallback), custom output dir (non-iCloud), notifyOnEmpty
   - iPhone capture: link to SHORTCUT.md + 2-line summary
   - scheduling: `schedule install`, missed-run/wake semantics (DESIGN §11.3 user-facing wording), optional `pmset repeat wakeorpoweron` manual tip (explicitly "optional, not managed by earmark")
   - listening guide: where files land, players with chapter support (QuickTime, Apple Books import; iOS Files caveat — U5 findings), rollover/cap behavior explanation (P10)
   - troubleshooting: table keyed by doctor check id + common run failures (each user-visible error code with meaning and action — generate from DESIGN §12.2)
   - uninstall: `schedule uninstall`, `npm uninstall -g earmark`, list of dirs to remove (all §6 paths), stated as "no other **earmark-created** files remain" — external prerequisites (Homebrew packages, VOICEVOX, Ollama and its pulled models) are explicitly not removed
   - privacy/security section for users: network egress is exactly (1) fetching your own article URLs, (2) one-time local model downloads (Kokoro/Ollama pulls), (3) nothing else by default — plus the caveat that pointing provider base URLs at non-loopback hosts sends article text over the network with a startup warning (DESIGN §13.6); inbox trust note.
3. Execute the **real-world acceptance checklist** (ISSUE_PLAN §6.5) on the dev Mac and record results in the PR: ≥3 articles incl. 1 English + 1 paywall-failure, full run, listening check, chapter navigation check, next-morning launchd fire, rollover verification. Any defect found → file as bug (new issue docs first per process) — this issue's docs still merge if defects are non-doc.
4. Every command/output in docs must be copy-paste-verified against the built CLI (no invented flags — cross-check DESIGN §3 and `--help`), except commands explicitly labeled "after the first release".

## Acceptance Criteria

- [ ] README renders correctly on GitHub, Japanese-first, ≤ ~150 lines, all commands verified.
- [ ] SETUP covers: all prerequisites with working install commands, config reference complete vs DESIGN §4.1 (reviewer diffs key-by-key), troubleshooting covers all §12.2 codes and all doctor check ids.
- [ ] SHORTCUT.md linked from both docs; uninstall list matches DESIGN §6 exactly.
- [ ] Acceptance checklist executed with evidence (transcripts, `log --last --json`, screenshot of chapters) attached to PR; U3/U5 observations recorded in ISSUE_PLAN edits within the PR.
- [ ] No doc contradicts DESIGN; where behavior differs from docs during verification, the code/DESIGN discrepancy is filed, not papered over.

## Validation

Doc review + the executed checklist evidence in the PR. A second reader (agent or human) follows SETUP from scratch-simulation (checking each command exists) and signs off in review.

## Dependencies

09, 24, 26, 27 (documented behavior final).

## Non-goals

Marketing site/screencasts; English README (one-paragraph pointer only in v1); auto-generated config docs tooling; CHANGELOG (issue 31).

## Design References

DESIGN §1, §3, §4.1, §6, §11.3, §12.2, §15; ISSUE_PLAN §6.5, U3, U5; ADR-001; issues 09/27.
