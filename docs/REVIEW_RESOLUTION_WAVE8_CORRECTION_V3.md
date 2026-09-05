# Wave 8 concrete review-resolution correction v3

- Repository: `Saber5656/earmark`
- Pull request: #1
- Current PR head identity: the authoritative value is supplied by the immutable review manifest; this file intentionally omits the mutable commit SHA to avoid a self-referential identity.
- Current PR base pinned for this review: `1e65839ee5cc9a91516d1b3880abc1dcb146c261`
- Previous correction artifact blob SHA: `2b04d99d2337f862e38470180ea4b80fba816c71`
- This v3 artifact supersedes the earlier generic resolution addenda for the exact threads below.
- The immutable review manifest pins the current head, current base, and correction-file blob identity for this review; any later change invalidates this evidence and requires a fresh review.
- This is documentation-level handling for documentation-only PRs. It is not a claim that product implementation, runtime tests, build, CI, security, or release validation is complete.
- No PR review bot is re-triggered. The file's own current blob SHA is intentionally held only in the immutable review manifest because embedding it here would be self-referential.

## Per-repository blocking handoffs

| Repository | QA/full-validation handoff | Security/privacy handoff |
|---|---|---|
| `Saber5656/earmark#1` | `tech-qa`/`tech-tester` must execute API/schema, codec, TTS, lifecycle, launchd, queue, notification, docs-lint, and repository-full-validation gates for this head/base. | `tech-security`/`tech-devopssec` must accept credential URL, remote egress, temp-file, output, path, static-audit, and packaging boundaries. |
| `Saber5656/tracegit#1` | `tech-qa`/`tech-tester` must execute record/schema, parser, CLI, editor, note, sync, E2E, docs-lint, and repository-full-validation gates for this head/base. | `tech-security`/`tech-devopssec` must accept note-size, file-mode, ref-update, sanitization, output-bound, and Git execution boundaries. |
| `Saber5656/mcplay#1` | `tech-qa`/`tech-tester` must execute schema, recorder, replay, dependency, CLI, protocol, docs-lint, and repository-full-validation gates for this head/base. | `tech-security`/`tech-devopssec` must accept spawn, path, permission, redaction, embedded-command, export, persistence, and threat-model boundaries. |

These handoffs are blocking: missing, pending, failed, skipped, cancelled, timed-out, stale, or non-accepting specialist/full-validation evidence prevents thread resolution and merge.

## Thread contracts

### 1. Thread `PRRT_kwDOTNkHgM6QDeia` — Align the translation interface signature with DESIGN.md

- File: `docs/decisions/ADR-004-provider-abstractions.md`
- Line: 27
- Finding basis: Line 20 uses translate(paragraphs, from, to) , but DESIGN.md §8.5 and issue 15 specify translate(req) with { paragraphs, sourceLang, targetLang } . Document one canonical signature consistently across the ADR and implementation issue.

**Normative resolution**: At `docs/decisions/ADR-004-provider-abstractions.md:27`, the canonical contract SHALL accept only `translate(req)` with the object fields `{ paragraphs, sourceLang, targetLang }`; positional arguments and missing or empty required fields SHALL be rejected before provider invocation.

**Focused verification gate**: Compare DESIGN §8.5, ADR-004, and Issue 15; validate `{ paragraphs, sourceLang, targetLang }`, reject the positional/missing/empty variants, and assert invalid input cannot reach a provider.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 2. Thread `PRRT_kwDOTNkHgM6QDeib` — Require SQLite foreign-key enforcement explicitly

- File: `docs/DESIGN.md`
- Line: 331
- Finding basis: The schema declares REFERENCES , but the design never requires enabling SQLite foreign keys. Add PRAGMA foreign keys = ON for every database connection before migrations/queries; otherwise orphaned digest items rows can be created if parent rows are removed or corrupted.

**Normative resolution**: At `docs/DESIGN.md:331`, the canonical contract SHALL execute `PRAGMA foreign_keys = ON` before migrations or queries and SHALL abort initialization when the readback is not `1`.

**Focused verification gate**: Open a fresh DB connection, read back `PRAGMA foreign_keys`, run migrations, then attempt an orphan `digest_items` operation; require pragma `1` and rejection of the invalid relationship.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 3. Thread `PRRT_kwDOTNkHgM6QDeic` — Reject credential-bearing provider URLs

- File: `docs/issues/02-config-module.md`
- Line: 30
- Finding basis: Line [30] allows values such as http://user:password@host , which would persist credentials in config.json and conflict with DESIGN §13.4’s zero-secret requirement. Reject URL usernames/passwords before saving, with a regression test.

**Normative resolution**: At `docs/issues/02-config-module.md:30`, the canonical contract SHALL reject every provider URL containing a username or password before writing `config.json`, and SHALL leave the existing file unchanged on rejection.

**Focused verification gate**: Validate credential-bearing and safe URLs for both provider fields; assert the former fails before `config.json` changes and the error contains no username/password.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 4. Thread `PRRT_kwDOTNkHgM6QDeie` — Expose `fallbackTitle` in the extractor API

- File: `docs/issues/11-content-extractor.md`
- Line: 22
- Finding basis: The API omits the optional fallbackTitle parameter, but the title-precedence requirement depends on it. Add it to the input contract and cover the corresponding call path in tests; otherwise titles fall back directly to the hostname.

**Normative resolution**: At `docs/issues/11-content-extractor.md:22`, the canonical contract SHALL include optional `fallbackTitle`, with title precedence fixed as extracted title, then `fallbackTitle`, then hostname.

**Focused verification gate**: Run the title matrix for extracted title, fallback title, empty fallback, and hostname; assert the documented precedence and optional API field.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 5. Thread `PRRT_kwDOTNkHgM6QDeif` — Make delimiter escaping robust across the provider boundary

- File: `docs/issues/14-translation-pipeline.md`
- Line: 34
- Finding basis: Prefixing [[3]] with one leading space is not a reliable escape for the Ollama wire format: the provider may normalize that space, after which decodeBlocks interprets the content as a delimiter and raises BlockFormatError . Use a non-whitespace escape sequence or a structured/length-delimited encoding, and add an end-to-end test with a paragraph exactly matching [[3]] .

**Normative resolution**: At `docs/issues/14-translation-pipeline.md:34`, the canonical contract SHALL use UTF-8 length-prefixed frames for every paragraph, so delimiter-looking text including exactly `[[3]]` is data and cannot be parsed as a frame delimiter.

**Focused verification gate**: Round-trip a paragraph exactly equal to `[[3]]` through provider whitespace normalization and Unicode/newline cases; assert lossless output and `BlockFormatError` for truncated/invalid frames.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 6. Thread `PRRT_kwDOTNkHgM6QDeig` — Define typed handling for every non-success response

- File: `docs/issues/15-ollama-translation-provider.md`
- Line: 25
- Finding basis: The retry matrix covers network errors, 5xx responses, timeouts, and decoded contract failures, but it does not specify 4xx responses, non-JSON bodies, or missing message.content . Ensure these paths are classified before parsing and always produce the provider’s typed TRANSLATE UNAVAILABLE or TRANSLATE INVALID OUTPUT error rather than leaking a raw HTTP/JSON exception or bypassing the article retry path.

**Normative resolution**: At `docs/issues/15-ollama-translation-provider.md:25`, the canonical contract SHALL classify network errors, timeouts, and 5xx responses as `TRANSLATE UNAVAILABLE` with one article-level retry; it SHALL classify 4xx responses, non-JSON 2xx bodies, missing `message.content`, and decoded contract failures as `TRANSLATE INVALID OUTPUT` without exposing raw exceptions.

**Focused verification gate**: Exercise network/timeout, 4xx, 5xx, 2xx non-JSON, missing-content, and valid-response fixtures; assert the exact typed error, one retry, and no raw exception.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 7. Thread `PRRT_kwDOTNkHgM6QDeii` — Define atomic output and strict WAV validation in the shared contract

- File: `docs/issues/16-tts-interface.md`
- Line: 32
- Finding basis: VoicevoxProvider requires atomic writes, but the shared TtsProvider contract does not require every provider to leave outWavPath absent or complete on failure. Also specify rejection of negative/non-finite durations, zero or invalid sample rates, data sizes not divisible by the frame size, and truncated fmt / data chunks; otherwise corrupt files can produce incorrect chapter timing or partial outputs.

**Normative resolution**: At `docs/issues/16-tts-interface.md:32`, the canonical contract SHALL publish a complete WAV atomically and SHALL leave the final path absent on failure; duration SHALL be finite and non-negative, sample rate SHALL be a positive integer, data bytes SHALL be divisible by frame size, and `fmt` and `data` chunks SHALL be complete before timing is accepted.

**Focused verification gate**: Feed negative/non-finite duration, invalid sample-rate, frame-misaligned, and truncated WAV fixtures plus provider failure; assert `AUDIO_BAD_WAV`, no partial final path, and correct duration only for valid data.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 8. Thread `PRRT_kwDOTNkHgM6QDeij` — Require explicit opt-in for non-loopback HTTP engines

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding basis: The provider may send article text to any configured non-loopback http:// endpoint, while the product is otherwise local-first and zero-secret. A warning alone is easy to miss and does not establish a privacy boundary; require an explicit remote-engine opt-in, or reject non-loopback endpoints by default.

**Normative resolution**: At `docs/issues/17-voicevox-provider.md:18`, the canonical contract SHALL adopt this exact decision: The provider SHALL deny non-loopback HTTP by default and SHALL require a dedicated remote-engine opt-in flag before both availability checks and synthesis may contact it; loopback remains allowed.

**Focused verification gate**: Observe requests for loopback, remote-without-flag, and remote-with-flag cases; assert only the last remote case contacts the endpoint.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 9. Thread `PRRT_kwDOTNkHgM6QDeik` — Use one monotonic readiness deadline

- File: `docs/issues/18-voicevox-engine-lifecycle.md`
- Line: 35
- Finding basis: A 2-second health-request timeout combined with polling every second can make the “60 s” startup timeout substantially longer than 60 seconds. Compute a single deadline and cap each probe timeout by the remaining budget before killing the child and returning TTS UNAVAILABLE .

**Normative resolution**: At `docs/issues/18-voicevox-engine-lifecycle.md:35`, the canonical contract SHALL use one monotonic 60-second deadline; every health probe timeout SHALL be capped by the remaining budget, and expiry SHALL kill the child and return `TTS UNAVAILABLE`.

**Focused verification gate**: Use a health probe that consumes its timeout; assert one monotonic 60-second deadline caps every probe, kills the child at expiry, and returns `TTS_UNAVAILABLE`.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 10. Thread `PRRT_kwDOTNkHgM6QDeil` — Avoid `Promise.race` for synthesis timeout

- File: `docs/issues/19-kokoro-en-provider.md`
- Line: 38
- Finding basis: Promise.race only rejects the caller; it won’t stop kokoro-js generation, so a timed-out synthesis can keep burning CPU and overlap the next sequential request. Use a cancelable execution model here, such as a worker/process you can terminate on timeout.

**Normative resolution**: At `docs/issues/19-kokoro-en-provider.md:38`, the canonical contract SHALL run in a dedicated worker that is terminated at the 120-second deadline; a timed-out request SHALL produce no late output, partial file, or concurrent second synthesis.

**Focused verification gate**: Run a stuck Kokoro generation followed by a second request; assert worker termination at 120 seconds, no late output/partial file, and no overlap.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 11. Thread `PRRT_kwDOTNkHgM6QDeim` — Require private, exclusively-created temporary files

- File: `docs/issues/20-say-fallback-provider.md`
- Line: 24
- Finding basis: The temporary text file contains article content, but its permissions and creation mode are unspecified. Create it under a private work directory with exclusive creation and restrictive permissions, and retain the atomic output guarantee from the shared TTS contract.

**Normative resolution**: At `docs/issues/20-say-fallback-provider.md:24`, the canonical contract SHALL create a 0700 private work directory, create the article temp file with exclusive creation and mode 0600, and atomically publish the final WAV while deleting temporary material on every exit path.

**Focused verification gate**: Inspect concurrent temp-directory/file creation and modes 0700/0600, force a collision and failure, and assert exclusive refusal plus cleanup.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 12. Thread `PRRT_kwDOTNkHgM6QDein` — Resolve the chapter-gap timestamp contradiction

- File: `docs/issues/22-audio-assembly.md`
- Line: 30
- Finding basis: The text says the chapter gap “belongs to the previous chapter’s end,” but also defines the previous chapter’s endMs as the start of that gap. Those rules leave the gap outside both chapters. Specify one invariant explicitly—for example, whether END includes the following gap—and add hand-computed tests for both chapter boundaries.

**Normative resolution**: At `docs/issues/22-audio-assembly.md:30`, the canonical contract SHALL include the following gap: for every adjacent pair, `next.startMs` SHALL equal `previous.endMs`, and `previous.endMs` SHALL equal its audio end plus the following gap; the final chapter SHALL end at total duration.

**Focused verification gate**: Calculate three chapters with two non-zero gaps and a zero-gap control; assert `previous.endMs` includes each gap, `next.startMs` equals it, and total duration is consistent.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 13. Thread `PRRT_kwDOTNkHgM6QDeio` — Use an atomic no-clobber publish operation

- File: `docs/issues/22-audio-assembly.md`
- Line: 49
- Finding basis: Checking for the final path before fs.rename is a TOCTOU race: another process can create the target after the check, and POSIX rename can replace it. This violates the stated “never overwrites a final file” guarantee. Publish with an operation that fails atomically when the destination exists, and clean up the .part file on every failed publish attempt.

**Normative resolution**: At `docs/issues/22-audio-assembly.md:49`, the canonical contract SHALL publish a completed `.part` file by an exclusive same-directory hard link to the final path, then unlink `.part`; an existing destination SHALL make the link fail without replacement, followed by cleanup.

**Focused verification gate**: Race two publishers and pre-create the destination; assert one complete winner, no overwrite, collision failure, and no `.part` file.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 14. Thread `PRRT_kwDOTNkHgM6QDeiq` — Align `DryRunPlan` with the promised JSON contract

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding basis: The requirements and acceptance criteria promise translation: 'pending' for cross-language articles, but the declared DryRunPlan shape only contains needsTranslation . Add and define that field, or remove the promise from the requirements and tests.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:52`, the canonical contract SHALL adopt this exact decision: `DryRunPlan` SHALL add the enum field `translation` (`pending` for cross-language and `not-needed` otherwise) and reject missing, unknown, or inconsistent values.

**Focused verification gate**: Validate cross-language, same-language, missing, unknown-enum, and inconsistent `DryRunPlan` fixtures; assert `pending`/`not-needed` and rejection rules.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 15. Thread `PRRT_kwDOTNkHgM6QDeis` — Persist configuration only after successful bootstrap, or roll back on failure

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 25
- Finding basis: The documented sequence can write the new config and plist, boot out the existing agent, then fail during bootstrap . That leaves the single source of truth claiming the new schedule while no agent is loaded. Make install transactional: stage the plist, bootstrap successfully, then persist config; on failure restore the previous plist/config where possible.

**Normative resolution**: At `docs/issues/26-launchd-scheduling.md:25`, the canonical contract SHALL stage the new config and plist in a private directory, validate them, bootstrap the staged plist successfully, then atomically replace the previous config and plist; any failure SHALL remove staged files and restore the previous pair.

**Focused verification gate**: Inject launchd bootstrap failure after each staged write; assert prior plist/config bytes are restored and only a successfully loaded schedule is persisted.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 16. Thread `PRRT_kwDOTNkHgM6QDeit` — Include the optional config path in drift detection

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 27
- Finding basis: When install embeds --config <configPath , status currently checks only node/CLI paths and hour/minute. It can therefore report ok while launchd invokes a stale or missing configuration file. Compare the embedded config path with the current resolved path and verify its existence.

**Normative resolution**: At `docs/issues/26-launchd-scheduling.md:27`, the canonical contract SHALL compare the normalized absolute config path embedded in the launchd plist with the currently resolved path and SHALL report drift when they differ or when the resolved file is absent.

**Focused verification gate**: Compare identical, changed, relative, symlinked, and missing embedded config paths; assert only the existing exact resolved path reports `ok`.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 17. Thread `PRRT_kwDOTNkHgM6QDeiu` — Require a semantic static audit, not only text-pattern matching

- File: `docs/issues/29-security-boundary-tests.md`
- Line: 25
- Finding basis: This suite is a release gate for command execution and network boundaries, but the requirements do not require AST-aware analysis. A lexical scan can miss aliased or indirect calls such as const run = execFile , computed property access, multiline expressions, or dynamically constructed shell options. Require AST/semantic checks, or add negative fixtures covering these bypass forms, so the audit actually enforces the DESIGN §13 boundary.

**Normative resolution**: At `docs/issues/29-security-boundary-tests.md:25`, the canonical contract SHALL parse TypeScript and JavaScript with the TypeScript compiler API, resolve imported aliases and member/computed calls, and fail on forbidden execution or network paths; lexical matching SHALL be supplementary only.

**Focused verification gate**: Run semantic audit fixtures for direct, aliased, computed-property, multiline, and dynamic shell/network sinks; assert bypasses fail and analyzer errors cannot pass.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 18. Thread `PRRT_kwDOTNkHgM6QDeiv` — Scope the “exactly” network-egress promise to direct app traffic

- File: `docs/issues/30-setup-usage-docs.md`
- Line: 34
- Finding basis: The design includes iCloud-folder capture, whose synchronization is network activity even if earmark only reads the local folder. Distinguish direct app requests from OS-managed iCloud traffic so this user-facing privacy statement is not misleading.

**Normative resolution**: At `docs/issues/30-setup-usage-docs.md:34`, the canonical contract SHALL promise only that the application makes no direct network request during the local capture path and SHALL state separately that OS-managed iCloud synchronization may use the network.

**Focused verification gate**: Classify direct fetch/provider/telemetry, local-folder read, and OS-managed iCloud sync in the privacy table; assert the promise is limited to direct app traffic.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 19. Thread `PRRT_kwDOTNkHgM6QDeiw` — Put the packed binary on `PATH` before invoking it

- File: `docs/issues/31-packaging-npm-prep.md`
- Line: 34
- Finding basis: npm install -g --prefix "$TMP/prefix" installs the executable under $TMP/prefix/bin , which is not guaranteed to be in the shell’s PATH . Export PATH="$TMP/prefix/bin:$PATH" or invoke the absolute path before running the smoke checks.

**Normative resolution**: At `docs/issues/31-packaging-npm-prep.md:34`, the canonical contract SHALL prepend `"$TMP/prefix/bin"` to `PATH` before invoking the installed executable and SHALL assert the invocation resolves to that packaged binary.

**Focused verification gate**: Run packaging smoke tests with and without prefix bin in `PATH` and with a conflicting global binary; assert the packed executable’s identity is used.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 20. Thread `PRRT_kwDOTNkHgM6QDfwK` — Pass selected counts into queue stats

- File: `docs/issues/23-batch-selection-state.md`
- Line: 31
- Finding basis: When a run prepares any articles successfully, this helper cannot compute the documented queueRemaining because its inputs are only the DB and staged failures; issue 24 calls it before persistence, so the DB still counts the soon-to-be-digested selected articles as queued . For example, with 10 selected articles, 8 prepared successfully, 2 retryable failures, and 4 untouched backlog items, the closing narration/log summary should report 6 remaining, but this API has no way to subtract the 8 successes unless the selected/prepared count or IDs are passed in.

**Normative resolution**: At `docs/issues/23-batch-selection-state.md:31`, the canonical contract SHALL receive selected article IDs and prepared-success IDs and SHALL calculate `queueRemaining` as the current queued count minus the prepared-success count; retryable failures SHALL remain counted.

**Focused verification gate**: Run the 10-selected/8-prepared/2-retryable/4-untouched case and all-success/all-failure/empty controls; require queue remaining 6/4/14/0.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 21. Thread `PRRT_kwDOTNkHgM6QDfwN` — Relax layering rule for planned shared modules

- File: `docs/issues/04-cli-skeleton.md`
- Line: 37
- Finding basis: This lint rule would block several imports that later issues explicitly require, so implementation PRs will either fail lint or duplicate security/audio logic. In particular, issue 10 depends on importing the IP classifiers from src/capture/url.ts , and the TTS providers in issues 17/19/20 need the WAV helpers from src/audio/wav.ts ; neither is a types.ts interface or pipeline/ . Please carve out these shared module imports or move the shared helpers into core before enforcing the rule.

**Normative resolution**: At `docs/issues/04-cli-skeleton.md:37`, the canonical contract SHALL explicitly allow imports from `src/capture/url.ts` and `src/audio/wav.ts` into the providers that require them, while all other cross-layer imports remain failures.

**Focused verification gate**: Lint approved imports from `src/capture/url.ts` and `src/audio/wav.ts` plus a forbidden import; assert only the planned exception passes.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 22. Thread `PRRT_kwDOTNkHgM6QDfwO` — Add translation status to the dry-run contract

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding basis: The dry-run prose and acceptance criteria require cross-language articles to be reported with translation: 'pending' , but the DryRunPlan type declared here has no translation field. An implementation that follows this contract will either omit the required status and fail the later acceptance test, or add an undocumented field that downstream JSON consumers cannot rely on.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:52`, the canonical contract SHALL contain `translation: "pending" | "not-needed"`; cross-language articles SHALL serialize `"pending"`, and same-language articles SHALL serialize `"not-needed"`.

**Focused verification gate**: Validate the dry-run type/example/acceptance trio with cross-language and same-language fixtures; reject a missing `translation` field.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 23. Thread `PRRT_kwDOTNkHgM6QDfwQ` — Make the digest terminal commit atomic

- File: `docs/issues/24-run-orchestrator.md`
- Line: 46
- Finding basis: This splits the terminal persistence into createDigestWithItems followed by separate content updates and failure persistence, which leaves a crash/DB error window after the digest and digested transitions are committed but before staged failures are recorded. In that scenario the same-day rerun is skipped because the digest exists, while failed selected articles can remain unaccounted for in retry counts/run summaries; include the content updates and applyArticleFailure calls in the same terminal transaction or define recovery semantics.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:46`, the canonical contract SHALL write the digest, item transitions, content updates, and `applyArticleFailure` records in one database transaction; any error SHALL roll back the complete terminal update.

**Focused verification gate**: Inject failure after each terminal DB operation and reopen the DB; assert digest, content, state, and failure writes roll back together and retry is safe.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 24. Thread `PRRT_kwDOTNkHgM6QDfwR` — Cover the remaining special-use IPv6 ranges

- File: `docs/issues/05-url-normalization.md`
- Line: 37
- Finding basis: The IPv6 blocklist is not the full special-use set promised by the fetch policy; it omits ranges such as local-use NAT64 64:ff9b:1::/48 , 6to4 2002::/16 , and other IANA special-purpose allocations under 2001::/23 . With allowPrivateNetworks=false , those literals or DNS/redirect targets would be classified as public and allowed through the SSRF guard despite the design goal of blocking every non-public/special target.

**Normative resolution**: At `docs/issues/05-url-normalization.md:37`, the canonical contract SHALL apply a checked-in CIDR table of all IANA special-purpose IPv6 allocations, including `64:ff9b:1::/48`, `2002::/16`, and the relevant `2001::/23` ranges, to literal addresses and every DNS or redirect target before connection.

**Focused verification gate**: Test the named IPv6 ranges in literals, DNS answers, and redirects plus public/unknown controls; assert every special-use/unknown target is blocked.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 25. Thread `PRRT_kwDOTNkHgM6QDfwS` — Resolve the empty codec test contradiction

- File: `docs/issues/14-translation-pipeline.md`
- Line: 55
- Finding basis: This property test asks for 0-paragraph arrays to round-trip, but the very next acceptance criterion requires zero delimiters to throw BlockFormatError . Either implementation choice fails one of the required tests ( decodeBlocks('', 0) → [] passes the property but fails the failure table, while throwing does the reverse), so the issue should either generate 1–4 paragraphs or explicitly define the zero-count case and adjust the failure table.

**Normative resolution**: At `docs/issues/14-translation-pipeline.md:55`, the canonical contract SHALL adopt this exact decision: `decodeBlocks("", 0)` SHALL return `[]`; empty input with a positive expected paragraph count SHALL raise `BlockFormatError`; the property generator SHALL use 1–4 paragraphs plus the explicit zero case.

**Focused verification gate**: Call `decodeBlocks("",0)` and `decodeBlocks("",1)` and run the 1–4 paragraph property corpus; require only the first returns `[]`.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 26. Thread `PRRT_kwDOTNkHgM6QDfwT` — Include fallbackTitle in the extractor API

- File: `docs/issues/11-content-extractor.md`
- Line: 17
- Finding basis: The extractor API omits the fallbackTitle parameter that the title-precedence requirement depends on. When a page has no usable extracted title, the run pipeline cannot pass the capture-time title from earmark add /inbox through this contract, so implementations following the API will fall back to the hostname and fail the documented precedence matrix.

**Normative resolution**: At `docs/issues/11-content-extractor.md:17`, the canonical contract SHALL carry optional `fallbackTitle` from capture through the run pipeline, and the documented precedence SHALL be extracted title, then fallback title, then hostname.

**Focused verification gate**: Trace fallback title from add/inbox to extraction with usable/empty/missing variants; assert the API, matrix, and output use the same precedence.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 27. Thread `PRRT_kwDOTNkHgM6QDfwU` — Distinguish VOICEVOX port squatting in availability

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding basis: This availability contract collapses wrong-shape /version responses into the same generic unavailable result as a stopped engine, but doctor later treats an unavailable engine plus a discoverable binary as autostart-ready. If another local service is already bound to port 50021, doctor can therefore report PASS even though issue 18 says autostart must not be attempted on a squatted port; return an explicit unexpected service state here and cover that path in doctor.

**Normative resolution**: At `docs/issues/17-voicevox-provider.md:18`, the canonical contract SHALL adopt this exact decision: A wrong-shaped `/version` response SHALL return `unexpected_service`; `doctor` SHALL refuse autostart in that state even when a VOICEVOX binary is discoverable.

**Focused verification gate**: Probe healthy, stopped, wrong-shaped, and foreign-service port cases with a discoverable binary; assert only stopped can autostart.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 28. Thread `PRRT_kwDOTNkHgM6QDfwX` — Avoid saying the queue is empty after all articles fail

- File: `docs/issues/25-notifications-run-log.md`
- Line: 25
- Finding basis: Issue 24 emits no articles not only for an empty queue but also when a selected batch exists and every article fails article-scoped processing. In that latter case this notification tells the user the queue is empty even though the failed/retryable articles remain in the queue, so the message should distinguish selected === 0 from prepared === 0 with failures .

**Normative resolution**: At `docs/issues/25-notifications-run-log.md:25`, the canonical contract SHALL be emitted only when `selected === 0`; when `selected > 0` and `prepared === 0`, it SHALL report that all selected articles failed and that retryable articles remain queued.

**Focused verification gate**: Render empty-selection, partial-success, all-retryable-failure, and permanent-failure notifications; assert only selected-zero uses `no_articles`.

**Completion boundary**: This contract is design-level only. Resolve the GitHub thread only after the focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

## Merge boundary

- `gate-task-evaluator` must re-fetch PR state, current head/base, required-check inventory, review decision, unresolved thread state, policy version, and merge candidate immediately before any merge mutation.
- `github_mergeable` or a successful CodeRabbit status is not merge authorization.
- The current task instruction allows at most one PR Bot review; this artifact authorizes no Bot trigger or rerun.