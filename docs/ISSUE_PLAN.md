# earmark — v1 Issue Plan

- Status: Approved plan derived from `docs/DESIGN.md` (canonical). GitHub Issues are generated from `docs/issues/NN-*.md`; if they diverge, these files win.
- Every issue is sized for one focused task by a lower-capability implementation agent, with no cross-issue guessing.

## 1. v1 completion statement

v1 is complete when **all 31 issues below are implemented and their Validation sections pass**, producing this end state:

> On a macOS machine, a user can `npm install -g earmark` (or run from a checkout), run `earmark doctor` and follow its remediation hints (install ffmpeg, VOICEVOX, Ollama + model, iOS Shortcut), capture articles via `earmark add <url>` and the iPhone Share Sheet, and from then on receive each morning at 06:00 (on wake, if the Mac was asleep; a powered-off/logged-out Mac skips that morning) a single `earmark-YYYY-MM-DD.m4a` in their iCloud Drive `earmark/digests/` folder — containing an opening, one chapter per article (oldest-first, max 10, translated to the configured `ja`/`en` output language, read in full by local TTS), and a closing that reports failures and remaining queue — with queue management (`list/remove/requeue`), run history (`log`), macOS notifications, basic tests green in CI, and setup documentation.

Anything not covered by an issue below is a non-goal for v1 (DESIGN §1.4) or a known unknown (§8 below).

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-project-scaffold.md` | Project scaffold: TypeScript package, lint/test toolchain, CI | 0 |
| 02 | `issues/02-config-module.md` | Config module: schema, defaults, atomic persistence, `earmark config` | 0 |
| 03 | `issues/03-sqlite-storage.md` | SQLite storage: schema migration, repositories, state transitions | 0 |
| 04 | `issues/04-cli-skeleton.md` | CLI skeleton: command wiring, logger, error handler, exit codes | 0 |
| 05 | `issues/05-url-normalization.md` | URL validation & normalization library | 1 |
| 06 | `issues/06-add-command.md` | `earmark add` command | 1 |
| 07 | `issues/07-queue-commands.md` | Queue commands: `list`, `remove`, `requeue` | 1 |
| 08 | `issues/08-inbox-ingest.md` | iCloud inbox ingest module and `earmark ingest` | 1 |
| 09 | `issues/09-ios-shortcut-doc.md` | iOS Shortcut recipe documentation (`docs/SHORTCUT.md`) | 1 |
| 10 | `issues/10-http-fetcher.md` | Hardened HTTP fetcher | 2 |
| 11 | `issues/11-content-extractor.md` | Content extractor (defuddle + readability fallback) | 2 |
| 12 | `issues/12-speechify.md` | Speechify: HTML → speakable paragraphs | 2 |
| 13 | `issues/13-language-detection.md` | Language detection | 2 |
| 14 | `issues/14-translation-pipeline.md` | Translation interface, need-decision, chunker | 2 |
| 15 | `issues/15-ollama-translation-provider.md` | Ollama translation provider | 2 |
| 16 | `issues/16-tts-interface.md` | TTS provider interface, utterance splitter, WAV contract | 3 |
| 17 | `issues/17-voicevox-provider.md` | VOICEVOX TTS provider (ja) | 3 |
| 18 | `issues/18-voicevox-engine-lifecycle.md` | VOICEVOX engine lifecycle (discover/autostart/stop) | 3 |
| 19 | `issues/19-kokoro-en-provider.md` | Kokoro TTS provider (en) | 3 |
| 20 | `issues/20-say-fallback-provider.md` | macOS `say` fallback TTS provider (en) | 3 |
| 21 | `issues/21-script-builder.md` | Digest script builder (verbatim generator + i18n templates) | 3 |
| 22 | `issues/22-audio-assembly.md` | Audio assembly: ffmpeg concat, loudnorm, m4a chapters, tags | 3 |
| 23 | `issues/23-batch-selection-state.md` | Batch selection & article retry/state policy | 4 |
| 24 | `issues/24-run-orchestrator.md` | `earmark run` orchestrator (lock, dry-run, idempotency) | 4 |
| 25 | `issues/25-notifications-run-log.md` | macOS notifications & `earmark log` | 4 |
| 26 | `issues/26-launchd-scheduling.md` | launchd scheduling: `earmark schedule install/uninstall/status` | 4 |
| 27 | `issues/27-doctor-command.md` | `earmark doctor` environment diagnosis | 4 |
| 28 | `issues/28-e2e-integration-tests.md` | E2E pipeline test with mock engines | 5 |
| 29 | `issues/29-security-boundary-tests.md` | Security boundary test suite | 5 |
| 30 | `issues/30-setup-usage-docs.md` | Setup & usage documentation (README, SETUP) | 5 |
| 31 | `issues/31-packaging-npm-prep.md` | npm packaging & publish-readiness checklist | 5 |

## 3. Dependency table

`A ← B` means B must be merged before A starts. "soft" = interface/fixture reuse only, can be developed against the merged interface issue.

| Issue | Hard dependencies | Notes |
|---|---|---|
| 01 | — | |
| 02 | 01 | |
| 03 | 01, 02 | `EarmarkError`, config types |
| 04 | 01, 02 | logger + config load at startup |
| 05 | 01 | pure library |
| 06 | 03, 04, 05 | |
| 07 | 03, 04 | |
| 08 | 02, 03, 04, 05 | inbox paths from config |
| 09 | 08 | documents the frozen contract |
| 10 | 02, 04, 05 | logger contract from 04; IP classifiers from 05 |
| 11 | 01, 04 | `stripControl` from 04; input is an HTML string, independent of fetcher |
| 12 | 04, 11 | sanitizer from 04; consumes extractor output contract + committed fixture |
| 13 | 12 | detects on speakable text |
| 14 | 02, 12, 13 | chunker splits long paragraphs via `splitSentences` (12) |
| 15 | 02, 04, 14 | implements interface from 14; codec from 14 |
| 16 | 02, 12, 14 | `splitSentences` (core/textseg, 12), `ProviderHealth` (core/provider, 14) |
| 17 | 16 | |
| 18 | 02, 04, 17 | wires provider prepare/dispose; exports `discoverEngineBinary` for 27 |
| 19 | 02, 04, 16 | `writePcm16Wav` from 16 |
| 20 | 02, 16 | needs ffmpeg path from config for AIFF→WAV |
| 21 | 02, 12, 14 | consumes PreparedArticle model |
| 22 | 02, 16, 21 | chapter plan model from 21, WAV contract from 16 |
| 23 | 03 | |
| 24 | 04, 08, 10, 11, 12, 13, 14, 15, 16, 17, 18, 21, 22, 23 | en path unit-tested with a fake provider; real en providers land in 19/20 and e2e scenario D |
| 25 | 24 | renders run summary produced by 24 |
| 26 | 04, 24 | plist invokes `run --trigger launchd` |
| 27 | 15, 18, 19, 20, 26 | aggregates every `checkAvailability` + env checks |
| 28 | 20, 24 | mocks from 15/17; scenario D uses the `say` provider (20) |
| 29 | 05, 06, 07, 08, 10, 12, 24, 25 | table-driven tests over merged boundary modules + lock/notification surfaces |
| 30 | 09, 24, 26, 27 | documents final behavior |
| 31 | 28, 30 | ships only after CI + docs are green |

## 4. Implementation waves

| Wave | Issues | Parallelism | Gate to next wave |
|---|---|---|---|
| 0 Foundation | 01 → 02 → {03, 04} | 03∥04 | CI green; `earmark --version`, `config`, empty DB migrate |
| 1 Capture | 05 → {06, 07, 08} → 09 | 06∥07∥08 | `add`/`list`/`ingest` work against a real iCloud folder |
| 2 Content | {10, 11} → 12 → 13 → 14 → 15 | 10∥11 | fixture article → translated speakable paragraphs (mock + real Ollama spot check) |
| 3 Audio | 16 → {17+18, 19, 20} ∥ 21 → 22 | providers parallel | fixture script → valid chaptered m4a (mock TTS in CI, real VOICEVOX spot check) |
| 4 Orchestration | 23 → 24 → {25, 26} → 27 | 25∥26 | full `earmark run` on the dev Mac produces a real digest; schedule installs |
| 5 Release readiness | {28, 29} → 30 → 31 | 28∥29 | all validation strategy items (§6) pass |

## 5. Coverage table (DESIGN.md § → issues)

| DESIGN section | Covered by |
|---|---|
| §1 overview, scope, non-goals | all issues; scope guarded by Non-goals section of each issue |
| §2.2 module layout / import rules | 01, 04 (enforced from then on); `core/text.ts` 04, `core/textseg.ts` 12, `core/provider.ts` 14 |
| §2.3 dependency set | 01, 31 |
| §3 CLI surface | 04 (skeleton), 02 (`config`), 06 (`add`), 07 (`list/remove/requeue`), 08 (`ingest`), 24 (`run`), 25 (`log`), 26 (`schedule`), 27 (`doctor`) |
| §4 configuration | 02 |
| §5 data model & state machine | 03, 23 |
| §6 filesystem layout, inbox contract | 02 (paths), 08, 09 |
| §7 URL validation/normalization | 05, 06 |
| §8.1 fetch | 10 |
| §8.2 extract | 11 |
| §8.3 speechify | 12 |
| §8.4 language detection | 13 |
| §8.5 translation | 14, 15 |
| §8.6 reading-time estimate | 12 |
| §9.1–9.2 script model & verbatim generator | 21 |
| §9.3 WAV contract | 16 |
| §9.4 assembly | 22 |
| §10.1 TTS interface & utterance sizing | 16 |
| §10.2 VOICEVOX provider | 17 |
| §10.3 engine lifecycle | 18 |
| §10.4 Kokoro | 19 |
| §10.5 say | 20 |
| §11.1 run sequence | 24 |
| §11.2 batch selection | 23 |
| §11.3 launchd | 26 |
| §11.4 locking | 24 |
| §11.5 idempotency | 23, 24 |
| §12 errors & retry | 23 (policy), 24 (surfacing), 04 (exit codes) |
| §13 security model | 05, 08, 10, 12 (inline controls); 29 (verification suite); 13.7 in 01, 31 |
| §14 observability | 04 (logger), 24 (run summary), 25 (log cmd, notify) |
| §15 doctor | 27 |
| §16 testing strategy | 01 (CI), each issue's Validation, 28, 29 |
| §17 performance budget | 24 (timings capture), 15 (throughput measurement) |
| §18 packaging | 31 |
| SHORTCUT / SETUP user docs | 09, 30 |

Prose-only behavior check: every user-visible behavior in DESIGN §1.1 core loop is owned by an issue (capture 06/08/09, morning pipeline 24, digest content 21/22, delivery paths 02/22, rollover 23, listening docs 30).

## 6. Whole-product validation strategy

1. **Per-issue**: each issue's Validation section (unit/integration tests + manual commands) must pass before its PR merges; CI (from issue 01) runs typecheck+lint+tests on macOS.
2. **Wave gates**: table in §4 — each wave ends with a runnable checkpoint on the dev Mac, evidence recorded in the PR.
3. **Mock-based E2E in CI** (issue 28): fixture articles → real m4a; ffprobe asserts chapters/duration/tags; no real network.
4. **Security regression suite** (issue 29): table-driven boundary tests (URL/SSRF/inbox/control-chars/execFile audit) kept green permanently.
5. **Real-world acceptance (pre-tag, manual)**: on the dev Mac with real VOICEVOX + Ollama: capture ≥3 articles incl. ≥1 English (translation) and ≥1 failure case (paywall), run `earmark run`, listen to the digest start-to-end, verify chapters in QuickTime/Apple Books, verify next-morning launchd fire and rollover behavior. Checklist embedded in issue 30's validation.
6. **Publish gate** (issue 31): `npm pack` inspection, `npm audit` clean, license + history scan per repository policy; actual `npm publish` is a manual owner action outside v1 issues.

## 7. Deferred v2 items (recorded, not planned)

LLM summary/“radio show” ScriptGenerator; translation cache by content hash; cloud translation/TTS providers (DeepL etc. — must keep zero-secret defaults per ADR-001); per-article episode mode; bookmarklet + local HTTP capture endpoint; import from Instapaper/Raindrop/Wallabag/RSS; paywall/JS rendering via headless browser; `.m4b` Apple Books variant; cover artwork; playback-position-aware requeue; Linux support; menu-bar app.

## 8. Known unknowns (may spawn new issues during implementation)

| # | Unknown | Trigger point | Fallback already designed |
|---|---|---|---|
| U1 | kokoro-js / onnxruntime-node compatibility with current Node (dev machine: 26) | issue 19 start | `say` provider (issue 20) |
| U2 | VOICEVOX engine binary path variance across install methods | issue 18 | config `enginePath` + doctor remediation |
| U3 | iCloud sync/`.icloud` materialization behavior in headless launchd context | issues 08/26 real-world gate | `brctl download` + defer-to-next-run + doctor WARN; `MaterializeDatalessFiles` plist key experiment (issue 26) |
| U4 | gemma3:12b translation throughput/quality on the dev Mac (budget §17) | issue 15 measurement | documented model downgrade (`gemma3:4b`), chunk tuning |
| U5 | Chapter display in target players AND single-pass ffmpeg chapter muxing (`-map_chapters`) | issue 22 checkpoint + manual validation | ffprobe-verified chapters are the acceptance floor; two-pass remux fallback specified; `.m4b` variant is v2 |
| U6 | franc accuracy on short/mixed-language articles | issue 13 fixtures | html-lang fallback + `und` policy (DESIGN §8.4) |
| U7 | Non-UTF-8 (Shift_JIS/EUC-JP) Japanese pages in the wild | issue 10/11 fixtures | add `iconv-lite` only if a real fixture demands it (DESIGN §8.1) |
| U8 | VOICEVOX speaker id 3 stability across engine versions | issue 17 | id read from `/speakers` in doctor; config override |

## 9. Process notes

- GitHub Issues are created from these files only after this plan and the drafts pass Codex review (repository workflow requirement). One GitHub Issue per file, English title = the file's Title.
- Each implementation PR references its issue and must include the issue's Validation evidence.
- New scope discovered mid-implementation gets a new `docs/issues/NN-*.md` first (docs are canonical), then a GitHub Issue.
