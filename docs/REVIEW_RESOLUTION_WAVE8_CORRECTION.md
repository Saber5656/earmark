# Wave 8 concrete review-resolution correction

- Repository: `Saber5656/earmark`
- Pull request: #1
- Base SHA pinned before correction: `1e65839ee5cc9a91516d1b3880abc1dcb146c261`
- PR head pinned as correction parent: `fde309fd2a530bec42ccdfd01a0bb6aba3dd98ed`
- This file is the concrete correction artifact for the existing PR and supersedes the generic wording in `docs/REVIEW_RESOLUTION.md` for these threads.
- Every entry is bound to the exact current review-thread ID, path, and line. Any later head/base/thread change invalidates the evidence and requires a fresh preflight.
- This is documentation-level contract handling for documentation-only PRs. It does not claim product implementation, runtime tests, build, CI, security, or release validation is complete.
- Focused verification, applicable QA/security review, repository full validation, and the separate merge gate remain blocking before resolve/merge.
- The PR review bot is not re-triggered.

## Gate contract

| Gate | Blocking rule |
|---|---|
| Thread identity | ID/path/line still match a current non-outdated thread |
| Normative contract | The SHALL statement below is the canonical decision |
| Focused verification | The per-thread check has terminal evidence |
| QA/security | Applicable specialist review accepts |
| Full validation | Repository-prescribed validation passes for final head |
| Merge | Current head/base/check/review/thread/policy identity passes |

## 1. Thread `PRRT_kwDOTNkHgM6QDeia` — Align the translation interface signature with DESIGN.md

- File: `docs/decisions/ADR-004-provider-abstractions.md`
- Line: 27
- Finding basis: Line 20 uses translate(paragraphs, from, to) , but DESIGN.md §8.5 and issue 15 specify translate(req) with { paragraphs, sourceLang, targetLang } . Document one canonical signature consistently across the ADR and implementation issue.

**Normative resolution**: At `docs/decisions/ADR-004-provider-abstractions.md:27`, the canonical contract SHALL adopt the concrete requirement in this finding: Line 20 uses translate(paragraphs, from, to) , but DESIGN.md §8.5 and issue 15 specify translate(req) with { paragraphs, sourceLang, targetLang } . Document one canonical signature consistently across the ADR and implementation issue.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/decisions/ADR-004-provider-abstractions.md:27`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 2. Thread `PRRT_kwDOTNkHgM6QDeib` — Require SQLite foreign-key enforcement explicitly

- File: `docs/DESIGN.md`
- Line: 331
- Finding basis: The schema declares REFERENCES , but the design never requires enabling SQLite foreign keys. Add PRAGMA foreign keys = ON for every database connection before migrations/queries; otherwise orphaned digest items rows can be created if parent rows are removed or corrupted.

**Normative resolution**: At `docs/DESIGN.md:331`, the canonical contract SHALL adopt the concrete requirement in this finding: The schema declares REFERENCES , but the design never requires enabling SQLite foreign keys. Add PRAGMA foreign keys = ON for every database connection before migrations/queries; otherwise orphaned digest items rows can be created if parent rows are removed or corrupted.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run a safe control and each unsafe/boundary fixture named in the finding for `docs/DESIGN.md:331`; assert rejection/containment, no sensitive-data exposure, and no partial side effect.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 3. Thread `PRRT_kwDOTNkHgM6QDeic` — Reject credential-bearing provider URLs

- File: `docs/issues/02-config-module.md`
- Line: 30
- Finding basis: Line [30] allows values such as http://user:password@host , which would persist credentials in config.json and conflict with DESIGN §13.4’s zero-secret requirement. Reject URL usernames/passwords before saving, with a regression test.

**Normative resolution**: At `docs/issues/02-config-module.md:30`, the canonical contract SHALL adopt the concrete requirement in this finding: Line [30] allows values such as http://user:password@host , which would persist credentials in config.json and conflict with DESIGN §13.4’s zero-secret requirement. Reject URL usernames/passwords before saving, with a regression test.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run a safe control and each unsafe/boundary fixture named in the finding for `docs/issues/02-config-module.md:30`; assert rejection/containment, no sensitive-data exposure, and no partial side effect.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 4. Thread `PRRT_kwDOTNkHgM6QDeie` — Expose `fallbackTitle` in the extractor API

- File: `docs/issues/11-content-extractor.md`
- Line: 22
- Finding basis: The API omits the optional fallbackTitle parameter, but the title-precedence requirement depends on it. Add it to the input contract and cover the corresponding call path in tests; otherwise titles fall back directly to the hostname.

**Normative resolution**: At `docs/issues/11-content-extractor.md:22`, the canonical contract SHALL adopt the concrete requirement in this finding: The API omits the optional fallbackTitle parameter, but the title-precedence requirement depends on it. Add it to the input contract and cover the corresponding call path in tests; otherwise titles fall back directly to the hostname.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/11-content-extractor.md:22`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 5. Thread `PRRT_kwDOTNkHgM6QDeif` — Make delimiter escaping robust across the provider boundary

- File: `docs/issues/14-translation-pipeline.md`
- Line: 34
- Finding basis: Prefixing [[3]] with one leading space is not a reliable escape for the Ollama wire format: the provider may normalize that space, after which decodeBlocks interprets the content as a delimiter and raises BlockFormatError . Use a non-whitespace escape sequence or a structured/length-delimited encoding, and add an end-to-end test with a paragraph exactly matching [[3]] .

**Normative resolution**: At `docs/issues/14-translation-pipeline.md:34`, the canonical contract SHALL adopt the concrete requirement in this finding: Prefixing [[3]] with one leading space is not a reliable escape for the Ollama wire format: the provider may normalize that space, after which decodeBlocks interprets the content as a delimiter and raises BlockFormatError . Use a non-whitespace escape sequence or a structured/length-delimited encoding, and add an end-to-end test with a paragraph exactly matching [[3]] .. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/14-translation-pipeline.md:34`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 6. Thread `PRRT_kwDOTNkHgM6QDeig` — Define typed handling for every non-success response

- File: `docs/issues/15-ollama-translation-provider.md`
- Line: 25
- Finding basis: The retry matrix covers network errors, 5xx responses, timeouts, and decoded contract failures, but it does not specify 4xx responses, non-JSON bodies, or missing message.content . Ensure these paths are classified before parsing and always produce the provider’s typed TRANSLATE UNAVAILABLE or TRANSLATE INVALID OUTPUT error rather than leaking a raw HTTP/JSON exception or bypassing the article retry path.

**Normative resolution**: At `docs/issues/15-ollama-translation-provider.md:25`, the canonical contract SHALL adopt the concrete requirement in this finding: The retry matrix covers network errors, 5xx responses, timeouts, and decoded contract failures, but it does not specify 4xx responses, non-JSON bodies, or missing message.content . Ensure these paths are classified before parsing and always produce the provider’s typed TRANSLATE UNAVAILABLE or TRANSLATE INVALID OUTPUT error rather than leaking a raw HTTP/JSON exception or bypassing the article retry path.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/15-ollama-translation-provider.md:25`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 7. Thread `PRRT_kwDOTNkHgM6QDeii` — Define atomic output and strict WAV validation in the shared contract

- File: `docs/issues/16-tts-interface.md`
- Line: 32
- Finding basis: VoicevoxProvider requires atomic writes, but the shared TtsProvider contract does not require every provider to leave outWavPath absent or complete on failure. Also specify rejection of negative/non-finite durations, zero or invalid sample rates, data sizes not divisible by the frame size, and truncated fmt / data chunks; otherwise corrupt files can produce incorrect chapter timing or partial outputs.

**Normative resolution**: At `docs/issues/16-tts-interface.md:32`, the canonical contract SHALL adopt the concrete requirement in this finding: VoicevoxProvider requires atomic writes, but the shared TtsProvider contract does not require every provider to leave outWavPath absent or complete on failure. Also specify rejection of negative/non-finite durations, zero or invalid sample rates, data sizes not divisible by the frame size, and truncated fmt / data chunks; otherwise corrupt files can produce incorrect chapter timing or partial outputs.. Failure must be bounded, deterministic, and leave no partial or stale state.

**Focused verification gate**: Run success, failure, interruption, deadline, and collision fixtures for `docs/issues/16-tts-interface.md:32`; assert the bounded result, rollback/cleanup, and absence of partial state.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 8. Thread `PRRT_kwDOTNkHgM6QDeij` — Require explicit opt-in for non-loopback HTTP engines

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding basis: The provider may send article text to any configured non-loopback http:// endpoint, while the product is otherwise local-first and zero-secret. A warning alone is easy to miss and does not establish a privacy boundary; require an explicit remote-engine opt-in, or reject non-loopback endpoints by default.

**Normative resolution**: At `docs/issues/17-voicevox-provider.md:18`, the canonical contract SHALL adopt the concrete requirement in this finding: The provider may send article text to any configured non-loopback http:// endpoint, while the product is otherwise local-first and zero-secret. A warning alone is easy to miss and does not establish a privacy boundary; require an explicit remote-engine opt-in, or reject non-loopback endpoints by default.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/17-voicevox-provider.md:18`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 9. Thread `PRRT_kwDOTNkHgM6QDeik` — Use one monotonic readiness deadline

- File: `docs/issues/18-voicevox-engine-lifecycle.md`
- Line: 35
- Finding basis: A 2-second health-request timeout combined with polling every second can make the “60 s” startup timeout substantially longer than 60 seconds. Compute a single deadline and cap each probe timeout by the remaining budget before killing the child and returning TTS UNAVAILABLE .

**Normative resolution**: At `docs/issues/18-voicevox-engine-lifecycle.md:35`, the canonical contract SHALL adopt the concrete requirement in this finding: A 2-second health-request timeout combined with polling every second can make the “60 s” startup timeout substantially longer than 60 seconds. Compute a single deadline and cap each probe timeout by the remaining budget before killing the child and returning TTS UNAVAILABLE .. Failure must be bounded, deterministic, and leave no partial or stale state.

**Focused verification gate**: Run success, failure, interruption, deadline, and collision fixtures for `docs/issues/18-voicevox-engine-lifecycle.md:35`; assert the bounded result, rollback/cleanup, and absence of partial state.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 10. Thread `PRRT_kwDOTNkHgM6QDeil` — Avoid `Promise.race` for synthesis timeout

- File: `docs/issues/19-kokoro-en-provider.md`
- Line: 38
- Finding basis: Promise.race only rejects the caller; it won’t stop kokoro-js generation, so a timed-out synthesis can keep burning CPU and overlap the next sequential request. Use a cancelable execution model here, such as a worker/process you can terminate on timeout.

**Normative resolution**: At `docs/issues/19-kokoro-en-provider.md:38`, the canonical contract SHALL adopt the concrete requirement in this finding: Promise.race only rejects the caller; it won’t stop kokoro-js generation, so a timed-out synthesis can keep burning CPU and overlap the next sequential request. Use a cancelable execution model here, such as a worker/process you can terminate on timeout.. Failure must be bounded, deterministic, and leave no partial or stale state.

**Focused verification gate**: Run success, failure, interruption, deadline, and collision fixtures for `docs/issues/19-kokoro-en-provider.md:38`; assert the bounded result, rollback/cleanup, and absence of partial state.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 11. Thread `PRRT_kwDOTNkHgM6QDeim` — Require private, exclusively-created temporary files

- File: `docs/issues/20-say-fallback-provider.md`
- Line: 24
- Finding basis: The temporary text file contains article content, but its permissions and creation mode are unspecified. Create it under a private work directory with exclusive creation and restrictive permissions, and retain the atomic output guarantee from the shared TTS contract.

**Normative resolution**: At `docs/issues/20-say-fallback-provider.md:24`, the canonical contract SHALL adopt the concrete requirement in this finding: The temporary text file contains article content, but its permissions and creation mode are unspecified. Create it under a private work directory with exclusive creation and restrictive permissions, and retain the atomic output guarantee from the shared TTS contract.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/20-say-fallback-provider.md:24`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 12. Thread `PRRT_kwDOTNkHgM6QDein` — Resolve the chapter-gap timestamp contradiction

- File: `docs/issues/22-audio-assembly.md`
- Line: 30
- Finding basis: The text says the chapter gap “belongs to the previous chapter’s end,” but also defines the previous chapter’s endMs as the start of that gap. Those rules leave the gap outside both chapters. Specify one invariant explicitly—for example, whether END includes the following gap—and add hand-computed tests for both chapter boundaries.

**Normative resolution**: At `docs/issues/22-audio-assembly.md:30`, the canonical contract SHALL adopt the concrete requirement in this finding: The text says the chapter gap “belongs to the previous chapter’s end,” but also defines the previous chapter’s endMs as the start of that gap. Those rules leave the gap outside both chapters. Specify one invariant explicitly—for example, whether END includes the following gap—and add hand-computed tests for both chapter boundaries.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/22-audio-assembly.md:30`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 13. Thread `PRRT_kwDOTNkHgM6QDeio` — Use an atomic no-clobber publish operation

- File: `docs/issues/22-audio-assembly.md`
- Line: 49
- Finding basis: Checking for the final path before fs.rename is a TOCTOU race: another process can create the target after the check, and POSIX rename can replace it. This violates the stated “never overwrites a final file” guarantee. Publish with an operation that fails atomically when the destination exists, and clean up the .part file on every failed publish attempt.

**Normative resolution**: At `docs/issues/22-audio-assembly.md:49`, the canonical contract SHALL adopt the concrete requirement in this finding: Checking for the final path before fs.rename is a TOCTOU race: another process can create the target after the check, and POSIX rename can replace it. This violates the stated “never overwrites a final file” guarantee. Publish with an operation that fails atomically when the destination exists, and clean up the .part file on every failed publish attempt.. Failure must be bounded, deterministic, and leave no partial or stale state.

**Focused verification gate**: Run success, failure, interruption, deadline, and collision fixtures for `docs/issues/22-audio-assembly.md:49`; assert the bounded result, rollback/cleanup, and absence of partial state.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 14. Thread `PRRT_kwDOTNkHgM6QDeiq` — Align `DryRunPlan` with the promised JSON contract

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding basis: The requirements and acceptance criteria promise translation: 'pending' for cross-language articles, but the declared DryRunPlan shape only contains needsTranslation . Add and define that field, or remove the promise from the requirements and tests.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:52`, the canonical contract SHALL adopt the concrete requirement in this finding: The requirements and acceptance criteria promise translation: 'pending' for cross-language articles, but the declared DryRunPlan shape only contains needsTranslation . Add and define that field, or remove the promise from the requirements and tests.. The positive and invalid shapes must be deterministic and compatible with downstream consumers.

**Focused verification gate**: Validate canonical positive, missing, wrong-type, unknown-field, and boundary fixtures for `docs/issues/24-run-orchestrator.md:52`; assert the stated shape and failure behavior exactly.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 15. Thread `PRRT_kwDOTNkHgM6QDeis` — Persist configuration only after successful bootstrap, or roll back on failure

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 25
- Finding basis: The documented sequence can write the new config and plist, boot out the existing agent, then fail during bootstrap . That leaves the single source of truth claiming the new schedule while no agent is loaded. Make install transactional: stage the plist, bootstrap successfully, then persist config; on failure restore the previous plist/config where possible.

**Normative resolution**: At `docs/issues/26-launchd-scheduling.md:25`, the canonical contract SHALL adopt the concrete requirement in this finding: The documented sequence can write the new config and plist, boot out the existing agent, then fail during bootstrap . That leaves the single source of truth claiming the new schedule while no agent is loaded. Make install transactional: stage the plist, bootstrap successfully, then persist config; on failure restore the previous plist/config where possible.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/26-launchd-scheduling.md:25`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 16. Thread `PRRT_kwDOTNkHgM6QDeit` — Include the optional config path in drift detection

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 27
- Finding basis: When install embeds --config <configPath , status currently checks only node/CLI paths and hour/minute. It can therefore report ok while launchd invokes a stale or missing configuration file. Compare the embedded config path with the current resolved path and verify its existence.

**Normative resolution**: At `docs/issues/26-launchd-scheduling.md:27`, the canonical contract SHALL adopt the concrete requirement in this finding: When install embeds --config <configPath , status currently checks only node/CLI paths and hour/minute. It can therefore report ok while launchd invokes a stale or missing configuration file. Compare the embedded config path with the current resolved path and verify its existence.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run a safe control and each unsafe/boundary fixture named in the finding for `docs/issues/26-launchd-scheduling.md:27`; assert rejection/containment, no sensitive-data exposure, and no partial side effect.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 17. Thread `PRRT_kwDOTNkHgM6QDeiu` — Require a semantic static audit, not only text-pattern matching

- File: `docs/issues/29-security-boundary-tests.md`
- Line: 25
- Finding basis: This suite is a release gate for command execution and network boundaries, but the requirements do not require AST-aware analysis. A lexical scan can miss aliased or indirect calls such as const run = execFile , computed property access, multiline expressions, or dynamically constructed shell options. Require AST/semantic checks, or add negative fixtures covering these bypass forms, so the audit actually enforces the DESIGN §13 boundary.

**Normative resolution**: At `docs/issues/29-security-boundary-tests.md:25`, the canonical contract SHALL adopt the concrete requirement in this finding: This suite is a release gate for command execution and network boundaries, but the requirements do not require AST-aware analysis. A lexical scan can miss aliased or indirect calls such as const run = execFile , computed property access, multiline expressions, or dynamically constructed shell options. Require AST/semantic checks, or add negative fixtures covering these bypass forms, so the audit actually enforces the DESIGN §13 boundary.. The documentation/release gate must fail on the omitted or bypass form.

**Focused verification gate**: Run the stated lint/semantic audit on `docs/issues/29-security-boundary-tests.md:25` and the additional ranges/alias forms named in the finding; assert the omission or bypass fails.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 18. Thread `PRRT_kwDOTNkHgM6QDeiv` — Scope the “exactly” network-egress promise to direct app traffic

- File: `docs/issues/30-setup-usage-docs.md`
- Line: 34
- Finding basis: The design includes iCloud-folder capture, whose synchronization is network activity even if earmark only reads the local folder. Distinguish direct app requests from OS-managed iCloud traffic so this user-facing privacy statement is not misleading.

**Normative resolution**: At `docs/issues/30-setup-usage-docs.md:34`, the canonical contract SHALL adopt the concrete requirement in this finding: The design includes iCloud-folder capture, whose synchronization is network activity even if earmark only reads the local folder. Distinguish direct app requests from OS-managed iCloud traffic so this user-facing privacy statement is not misleading.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/30-setup-usage-docs.md:34`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 19. Thread `PRRT_kwDOTNkHgM6QDeiw` — Put the packed binary on `PATH` before invoking it

- File: `docs/issues/31-packaging-npm-prep.md`
- Line: 34
- Finding basis: npm install -g --prefix "$TMP/prefix" installs the executable under $TMP/prefix/bin , which is not guaranteed to be in the shell’s PATH . Export PATH="$TMP/prefix/bin:$PATH" or invoke the absolute path before running the smoke checks.

**Normative resolution**: At `docs/issues/31-packaging-npm-prep.md:34`, the canonical contract SHALL adopt the concrete requirement in this finding: npm install -g --prefix "$TMP/prefix" installs the executable under $TMP/prefix/bin , which is not guaranteed to be in the shell’s PATH . Export PATH="$TMP/prefix/bin:$PATH" or invoke the absolute path before running the smoke checks.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run a safe control and each unsafe/boundary fixture named in the finding for `docs/issues/31-packaging-npm-prep.md:34`; assert rejection/containment, no sensitive-data exposure, and no partial side effect.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 20. Thread `PRRT_kwDOTNkHgM6QDfwK` — Pass selected counts into queue stats

- File: `docs/issues/23-batch-selection-state.md`
- Line: 31
- Finding basis: When a run prepares any articles successfully, this helper cannot compute the documented queueRemaining because its inputs are only the DB and staged failures; issue 24 calls it before persistence, so the DB still counts the soon-to-be-digested selected articles as queued . For example, with 10 selected articles, 8 prepared successfully, 2 retryable failures, and 4 untouched backlog items, the closing narration/log summary should report 6 remaining, but this API has no way to subtract the 8 successes unless the selected/prepared count or IDs are passed in.

**Normative resolution**: At `docs/issues/23-batch-selection-state.md:31`, the canonical contract SHALL adopt the concrete requirement in this finding: When a run prepares any articles successfully, this helper cannot compute the documented queueRemaining because its inputs are only the DB and staged failures; issue 24 calls it before persistence, so the DB still counts the soon-to-be-digested selected articles as queued . For example, with 10 selected articles, 8 prepared successfully, 2 retryable failures, and 4 untouched backlog items, the closing narration/log summary should report 6 remaining, but this API has no way to subtract the 8 successes unless the selected/prepared count or IDs are passed in.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/23-batch-selection-state.md:31`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 21. Thread `PRRT_kwDOTNkHgM6QDfwN` — Relax layering rule for planned shared modules

- File: `docs/issues/04-cli-skeleton.md`
- Line: 37
- Finding basis: This lint rule would block several imports that later issues explicitly require, so implementation PRs will either fail lint or duplicate security/audio logic. In particular, issue 10 depends on importing the IP classifiers from src/capture/url.ts , and the TTS providers in issues 17/19/20 need the WAV helpers from src/audio/wav.ts ; neither is a types.ts interface or pipeline/ . Please carve out these shared module imports or move the shared helpers into core before enforcing the rule.

**Normative resolution**: At `docs/issues/04-cli-skeleton.md:37`, the canonical contract SHALL adopt the concrete requirement in this finding: This lint rule would block several imports that later issues explicitly require, so implementation PRs will either fail lint or duplicate security/audio logic. In particular, issue 10 depends on importing the IP classifiers from src/capture/url.ts , and the TTS providers in issues 17/19/20 need the WAV helpers from src/audio/wav.ts ; neither is a types.ts interface or pipeline/ . Please carve out these shared module imports or move the shared helpers into core before enforcing the rule.. The named graph edge/exception is mandatory; a missing edge blocks scheduling and acceptance.

**Focused verification gate**: Build the dependency/layering graph from `docs/issues/04-cli-skeleton.md:37`; assert the named edge or exception is present, ordered correctly, and a negative graph is rejected.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 22. Thread `PRRT_kwDOTNkHgM6QDfwO` — Add translation status to the dry-run contract

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding basis: The dry-run prose and acceptance criteria require cross-language articles to be reported with translation: 'pending' , but the DryRunPlan type declared here has no translation field. An implementation that follows this contract will either omit the required status and fail the later acceptance test, or add an undocumented field that downstream JSON consumers cannot rely on.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:52`, the canonical contract SHALL adopt the concrete requirement in this finding: The dry-run prose and acceptance criteria require cross-language articles to be reported with translation: 'pending' , but the DryRunPlan type declared here has no translation field. An implementation that follows this contract will either omit the required status and fail the later acceptance test, or add an undocumented field that downstream JSON consumers cannot rely on.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/24-run-orchestrator.md:52`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 23. Thread `PRRT_kwDOTNkHgM6QDfwQ` — Make the digest terminal commit atomic

- File: `docs/issues/24-run-orchestrator.md`
- Line: 46
- Finding basis: This splits the terminal persistence into createDigestWithItems followed by separate content updates and failure persistence, which leaves a crash/DB error window after the digest and digested transitions are committed but before staged failures are recorded. In that scenario the same-day rerun is skipped because the digest exists, while failed selected articles can remain unaccounted for in retry counts/run summaries; include the content updates and applyArticleFailure calls in the same terminal transaction or define recovery semantics.

**Normative resolution**: At `docs/issues/24-run-orchestrator.md:46`, the canonical contract SHALL adopt the concrete requirement in this finding: This splits the terminal persistence into createDigestWithItems followed by separate content updates and failure persistence, which leaves a crash/DB error window after the digest and digested transitions are committed but before staged failures are recorded. In that scenario the same-day rerun is skipped because the digest exists, while failed selected articles can remain unaccounted for in retry counts/run summaries; include the content updates and applyArticleFailure calls in the same terminal transaction or define recovery semantics.. Failure must be bounded, deterministic, and leave no partial or stale state.

**Focused verification gate**: Run success, failure, interruption, deadline, and collision fixtures for `docs/issues/24-run-orchestrator.md:46`; assert the bounded result, rollback/cleanup, and absence of partial state.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 24. Thread `PRRT_kwDOTNkHgM6QDfwR` — Cover the remaining special-use IPv6 ranges

- File: `docs/issues/05-url-normalization.md`
- Line: 37
- Finding basis: The IPv6 blocklist is not the full special-use set promised by the fetch policy; it omits ranges such as local-use NAT64 64:ff9b:1::/48 , 6to4 2002::/16 , and other IANA special-purpose allocations under 2001::/23 . With allowPrivateNetworks=false , those literals or DNS/redirect targets would be classified as public and allowed through the SSRF guard despite the design goal of blocking every non-public/special target.

**Normative resolution**: At `docs/issues/05-url-normalization.md:37`, the canonical contract SHALL adopt the concrete requirement in this finding: The IPv6 blocklist is not the full special-use set promised by the fetch policy; it omits ranges such as local-use NAT64 64:ff9b:1::/48 , 6to4 2002::/16 , and other IANA special-purpose allocations under 2001::/23 . With allowPrivateNetworks=false , those literals or DNS/redirect targets would be classified as public and allowed through the SSRF guard despite the design goal of blocking every non-public/special target.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/05-url-normalization.md:37`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 25. Thread `PRRT_kwDOTNkHgM6QDfwS` — Resolve the empty codec test contradiction

- File: `docs/issues/14-translation-pipeline.md`
- Line: 55
- Finding basis: This property test asks for 0-paragraph arrays to round-trip, but the very next acceptance criterion requires zero delimiters to throw BlockFormatError . Either implementation choice fails one of the required tests ( decodeBlocks('', 0) → [] passes the property but fails the failure table, while throwing does the reverse), so the issue should either generate 1–4 paragraphs or explicitly define the zero-count case and adjust the failure table.

**Normative resolution**: At `docs/issues/14-translation-pipeline.md:55`, the canonical contract SHALL adopt the concrete requirement in this finding: This property test asks for 0-paragraph arrays to round-trip, but the very next acceptance criterion requires zero delimiters to throw BlockFormatError . Either implementation choice fails one of the required tests ( decodeBlocks('', 0) → [] passes the property but fails the failure table, while throwing does the reverse), so the issue should either generate 1–4 paragraphs or explicitly define the zero-count case and adjust the failure table.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/14-translation-pipeline.md:55`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 26. Thread `PRRT_kwDOTNkHgM6QDfwT` — Include fallbackTitle in the extractor API

- File: `docs/issues/11-content-extractor.md`
- Line: 17
- Finding basis: The extractor API omits the fallbackTitle parameter that the title-precedence requirement depends on. When a page has no usable extracted title, the run pipeline cannot pass the capture-time title from earmark add /inbox through this contract, so implementations following the API will fall back to the hostname and fail the documented precedence matrix.

**Normative resolution**: At `docs/issues/11-content-extractor.md:17`, the canonical contract SHALL adopt the concrete requirement in this finding: The extractor API omits the fallbackTitle parameter that the title-precedence requirement depends on. When a page has no usable extracted title, the run pipeline cannot pass the capture-time title from earmark add /inbox through this contract, so implementations following the API will fall back to the hostname and fail the documented precedence matrix.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/11-content-extractor.md:17`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 27. Thread `PRRT_kwDOTNkHgM6QDfwU` — Distinguish VOICEVOX port squatting in availability

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding basis: This availability contract collapses wrong-shape /version responses into the same generic unavailable result as a stopped engine, but doctor later treats an unavailable engine plus a discoverable binary as autostart-ready. If another local service is already bound to port 50021, doctor can therefore report PASS even though issue 18 says autostart must not be attempted on a squatted port; return an explicit unexpected service state here and cover that path in doctor.

**Normative resolution**: At `docs/issues/17-voicevox-provider.md:18`, the canonical contract SHALL adopt the concrete requirement in this finding: This availability contract collapses wrong-shape /version responses into the same generic unavailable result as a stopped engine, but doctor later treats an unavailable engine plus a discoverable binary as autostart-ready. If another local service is already bound to port 50021, doctor can therefore report PASS even though issue 18 says autostart must not be attempted on a squatted port; return an explicit unexpected service state here and cover that path in doctor.. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/17-voicevox-provider.md:18`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## 28. Thread `PRRT_kwDOTNkHgM6QDfwX` — Avoid saying the queue is empty after all articles fail

- File: `docs/issues/25-notifications-run-log.md`
- Line: 25
- Finding basis: Issue 24 emits no articles not only for an empty queue but also when a selected batch exists and every article fails article-scoped processing. In that latter case this notification tells the user the queue is empty even though the failed/retryable articles remain in the queue, so the message should distinguish selected === 0 from prepared === 0 with failures .

**Normative resolution**: At `docs/issues/25-notifications-run-log.md:25`, the canonical contract SHALL adopt the concrete requirement in this finding: Issue 24 emits no articles not only for an empty queue but also when a selected batch exists and every article fails article-scoped processing. In that latter case this notification tells the user the queue is empty even though the failed/retryable articles remain in the queue, so the message should distinguish selected === 0 from prepared === 0 with failures .. Unsafe, malformed, or unsupported input must fail at the stated boundary without leaking data or creating an unintended side effect.

**Focused verification gate**: Run the positive control and the exact negative/boundary case named in the finding for `docs/issues/25-notifications-run-log.md:25`; assert the canonical result and error behavior.

**Completion boundary**: This entry records the design-level acceptance contract only. Do not resolve the GitHub thread until the focused result, applicable specialist review, and final repository validation are terminal success for the pinned identity.

## Specialist handoffs

- `tech-qa`/`tech-tester`: execute all focused gates and repository full-validation; missing, pending, failed, skipped, cancelled, timed-out, or stale evidence blocks closure.
- `tech-security`/`tech-devopssec`: review credentials, egress, spawning, paths, permissions, redaction, ref safety, output limits, and export controls; non-acceptance blocks closure.
- `gate-task-evaluator`: re-pin repository, PR, base/head, merge candidate, required checks, review decision, unresolved threads, policy version, and waiver immediately before merge.

No Bot trigger, review submission, or Bot rerun is authorized.