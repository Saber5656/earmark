# Review resolution addendum

- Repository: `Saber5656/earmark`
- Pull request: #1
- Original PR head before this resolution addendum: `fb0f9c44dc367fb29b4ccce992fab825328dbfa5`
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates this evidence and requires a fresh review.
- Scope: each exact review thread below has a normative resolution and a focused verification gate.
- This is design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTNkHgM6QDeia` — Align the translation interface signature with DESIGN.md.

- File: `docs/decisions/ADR-004-provider-abstractions.md`
- Line: 27
- Finding summary: `Align the translation interface signature with DESIGN.md.`

**Normative resolution**: Revise the referenced contract so “Align the translation interface signature with DESIGN.md.” is represented by one canonical, schema-valid input/output shape; reconcile all linked sections and define validation and failure behavior.

**Focused verification gate**: Parse/validate the canonical example and exercise valid, invalid, missing, and extra-field/shape cases; assert all consumers observe the same documented contract.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 2. Thread `PRRT_kwDOTNkHgM6QDeib` — Require SQLite foreign-key enforcement explicitly.

- File: `docs/DESIGN.md`
- Line: 331
- Finding summary: `Require SQLite foreign-key enforcement explicitly.`

**Normative resolution**: Revise the referenced release/CI contract so “Require SQLite foreign-key enforcement explicitly.” is explicit, reproducible, and gated before publication; keep unresolved owner decisions as gates rather than silently choosing a value.

**Focused verification gate**: Run the release/CI dry-run and its negative gate cases from a clean checkout; assert the required pin, branch/event guard, artifact, and publication precondition.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 3. Thread `PRRT_kwDOTNkHgM6QDeic` — Reject credential-bearing provider URLs.

- File: `docs/issues/02-config-module.md`
- Line: 30
- Finding summary: `Reject credential-bearing provider URLs.`

**Normative resolution**: Revise the referenced contract so “Reject credential-bearing provider URLs.” is an explicit fail-closed security boundary: validate the input before use, preserve safe behavior, and reject or redact the unsafe case without leaking data or bypassing the stated policy.

**Focused verification gate**: Run focused positive and adversarial cases for the named boundary, including the unsafe input and a safe control; assert rejection/redaction, no side effect, and no sensitive data in output.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 4. Thread `PRRT_kwDOTNkHgM6QDeie` — Expose `fallbackTitle` in the extractor API.

- File: `docs/issues/11-content-extractor.md`
- Line: 22
- Finding summary: `Expose `fallbackTitle` in the extractor API.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Expose `fallbackTitle` in the extractor API.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 5. Thread `PRRT_kwDOTNkHgM6QDeif` — Make delimiter escaping robust across the provider boundary.

- File: `docs/issues/14-translation-pipeline.md`
- Line: 34
- Finding summary: `Make delimiter escaping robust across the provider boundary.`

**Normative resolution**: Revise the referenced contract so “Make delimiter escaping robust across the provider boundary.” is a normative boundedness/atomicity guarantee with one defined deadline or commit point, deterministic failure behavior, and cleanup of partial state.

**Focused verification gate**: Run focused success, timeout/limit, concurrent/partial-failure cases; assert the documented deadline/cap, deterministic error, cleanup, and no clobber or leaked state.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 6. Thread `PRRT_kwDOTNkHgM6QDeig` — Define typed handling for every non-success response.

- File: `docs/issues/15-ollama-translation-provider.md`
- Line: 25
- Finding summary: `Define typed handling for every non-success response.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Define typed handling for every non-success response.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 7. Thread `PRRT_kwDOTNkHgM6QDeii` — Define atomic output and strict WAV validation in the shared contract.

- File: `docs/issues/16-tts-interface.md`
- Line: 32
- Finding summary: `Define atomic output and strict WAV validation in the shared contract.`

**Normative resolution**: Revise the referenced contract so “Define atomic output and strict WAV validation in the shared contract.” is a normative boundedness/atomicity guarantee with one defined deadline or commit point, deterministic failure behavior, and cleanup of partial state.

**Focused verification gate**: Run focused success, timeout/limit, concurrent/partial-failure cases; assert the documented deadline/cap, deterministic error, cleanup, and no clobber or leaked state.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 8. Thread `PRRT_kwDOTNkHgM6QDeij` — Require explicit opt-in for non-loopback HTTP engines.

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding summary: `Require explicit opt-in for non-loopback HTTP engines.`

**Normative resolution**: Revise the referenced contract so “Require explicit opt-in for non-loopback HTTP engines.” is an explicit fail-closed security boundary: validate the input before use, preserve safe behavior, and reject or redact the unsafe case without leaking data or bypassing the stated policy.

**Focused verification gate**: Run focused positive and adversarial cases for the named boundary, including the unsafe input and a safe control; assert rejection/redaction, no side effect, and no sensitive data in output.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 9. Thread `PRRT_kwDOTNkHgM6QDeik` — Use one monotonic readiness deadline.

- File: `docs/issues/18-voicevox-engine-lifecycle.md`
- Line: 35
- Finding summary: `Use one monotonic readiness deadline.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Use one monotonic readiness deadline.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 10. Thread `PRRT_kwDOTNkHgM6QDeil` — Avoid `Promise.race` for synthesis timeout

- File: `docs/issues/19-kokoro-en-provider.md`
- Line: 38
- Finding summary: `Avoid `Promise.race` for synthesis timeout`

**Normative resolution**: Revise the referenced contract so “Avoid `Promise.race` for synthesis timeout” is a normative boundedness/atomicity guarantee with one defined deadline or commit point, deterministic failure behavior, and cleanup of partial state.

**Focused verification gate**: Run focused success, timeout/limit, concurrent/partial-failure cases; assert the documented deadline/cap, deterministic error, cleanup, and no clobber or leaked state.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 11. Thread `PRRT_kwDOTNkHgM6QDeim` — Require private, exclusively-created temporary files.

- File: `docs/issues/20-say-fallback-provider.md`
- Line: 24
- Finding summary: `Require private, exclusively-created temporary files.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Require private, exclusively-created temporary files.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 12. Thread `PRRT_kwDOTNkHgM6QDein` — Resolve the chapter-gap timestamp contradiction.

- File: `docs/issues/22-audio-assembly.md`
- Line: 30
- Finding summary: `Resolve the chapter-gap timestamp contradiction.`

**Normative resolution**: Revise the referenced contract so it normatively resolves “Resolve the chapter-gap timestamp contradiction.” and does not leave the reported ambiguity or failure mode to implementation choice.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 13. Thread `PRRT_kwDOTNkHgM6QDeio` — Use an atomic no-clobber publish operation.

- File: `docs/issues/22-audio-assembly.md`
- Line: 49
- Finding summary: `Use an atomic no-clobber publish operation.`

**Normative resolution**: Revise the referenced contract so “Use an atomic no-clobber publish operation.” is a normative boundedness/atomicity guarantee with one defined deadline or commit point, deterministic failure behavior, and cleanup of partial state.

**Focused verification gate**: Run focused success, timeout/limit, concurrent/partial-failure cases; assert the documented deadline/cap, deterministic error, cleanup, and no clobber or leaked state.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 14. Thread `PRRT_kwDOTNkHgM6QDeiq` — Align `DryRunPlan` with the promised JSON contract.

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding summary: `Align `DryRunPlan` with the promised JSON contract.`

**Normative resolution**: Revise the referenced contract so “Align `DryRunPlan` with the promised JSON contract.” is represented by one canonical, schema-valid input/output shape; reconcile all linked sections and define validation and failure behavior.

**Focused verification gate**: Parse/validate the canonical example and exercise valid, invalid, missing, and extra-field/shape cases; assert all consumers observe the same documented contract.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 15. Thread `PRRT_kwDOTNkHgM6QDeis` — Persist configuration only after successful bootstrap, or roll back on failure.

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 25
- Finding summary: `Persist configuration only after successful bootstrap, or roll back on failure.`

**Normative resolution**: Revise the referenced contract so it normatively resolves “Persist configuration only after successful bootstrap, or roll back on failure.” and does not leave the reported ambiguity or failure mode to implementation choice.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 16. Thread `PRRT_kwDOTNkHgM6QDeit` — Include the optional config path in drift detection.

- File: `docs/issues/26-launchd-scheduling.md`
- Line: 27
- Finding summary: `Include the optional config path in drift detection.`

**Normative resolution**: Revise the referenced contract so “Include the optional config path in drift detection.” is an explicit fail-closed security boundary: validate the input before use, preserve safe behavior, and reject or redact the unsafe case without leaking data or bypassing the stated policy.

**Focused verification gate**: Run focused positive and adversarial cases for the named boundary, including the unsafe input and a safe control; assert rejection/redaction, no side effect, and no sensitive data in output.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 17. Thread `PRRT_kwDOTNkHgM6QDeiu` — Require a semantic static audit, not only text-pattern matching.

- File: `docs/issues/29-security-boundary-tests.md`
- Line: 25
- Finding summary: `Require a semantic static audit, not only text-pattern matching.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Require a semantic static audit, not only text-pattern matching.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 18. Thread `PRRT_kwDOTNkHgM6QDeiv` — Scope the “exactly” network-egress promise to direct app traffic.

- File: `docs/issues/30-setup-usage-docs.md`
- Line: 34
- Finding summary: `Scope the “exactly” network-egress promise to direct app traffic.`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Scope the “exactly” network-egress promise to direct app traffic.”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 19. Thread `PRRT_kwDOTNkHgM6QDeiw` — Put the packed binary on `PATH` before invoking it.

- File: `docs/issues/31-packaging-npm-prep.md`
- Line: 34
- Finding summary: `Put the packed binary on `PATH` before invoking it.`

**Normative resolution**: Revise the referenced contract so “Put the packed binary on `PATH` before invoking it.” is an explicit fail-closed security boundary: validate the input before use, preserve safe behavior, and reject or redact the unsafe case without leaking data or bypassing the stated policy.

**Focused verification gate**: Run focused positive and adversarial cases for the named boundary, including the unsafe input and a safe control; assert rejection/redaction, no side effect, and no sensitive data in output.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 20. Thread `PRRT_kwDOTNkHgM6QDfwK` — Pass selected counts into queue stats

- File: `docs/issues/23-batch-selection-state.md`
- Line: 31
- Finding summary: `Pass selected counts into queue stats`

**Normative resolution**: Revise the referenced contract so it normatively resolves “Pass selected counts into queue stats” and does not leave the reported ambiguity or failure mode to implementation choice.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 21. Thread `PRRT_kwDOTNkHgM6QDfwN` — Relax layering rule for planned shared modules

- File: `docs/issues/04-cli-skeleton.md`
- Line: 37
- Finding summary: `Relax layering rule for planned shared modules`

**Normative resolution**: Revise the referenced contract so it normatively resolves “Relax layering rule for planned shared modules” and does not leave the reported ambiguity or failure mode to implementation choice.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 22. Thread `PRRT_kwDOTNkHgM6QDfwO` — Add translation status to the dry-run contract

- File: `docs/issues/24-run-orchestrator.md`
- Line: 52
- Finding summary: `Add translation status to the dry-run contract`

**Normative resolution**: Revise the referenced contract so “Add translation status to the dry-run contract” is represented by one canonical, schema-valid input/output shape; reconcile all linked sections and define validation and failure behavior.

**Focused verification gate**: Parse/validate the canonical example and exercise valid, invalid, missing, and extra-field/shape cases; assert all consumers observe the same documented contract.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 23. Thread `PRRT_kwDOTNkHgM6QDfwQ` — Make the digest terminal commit atomic

- File: `docs/issues/24-run-orchestrator.md`
- Line: 46
- Finding summary: `Make the digest terminal commit atomic`

**Normative resolution**: Revise the referenced contract so “Make the digest terminal commit atomic” is a normative boundedness/atomicity guarantee with one defined deadline or commit point, deterministic failure behavior, and cleanup of partial state.

**Focused verification gate**: Run focused success, timeout/limit, concurrent/partial-failure cases; assert the documented deadline/cap, deterministic error, cleanup, and no clobber or leaked state.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 24. Thread `PRRT_kwDOTNkHgM6QDfwR` — Cover the remaining special-use IPv6 ranges

- File: `docs/issues/05-url-normalization.md`
- Line: 37
- Finding summary: `Cover the remaining special-use IPv6 ranges`

**Normative resolution**: Revise the referenced release/CI contract so “Cover the remaining special-use IPv6 ranges” is explicit, reproducible, and gated before publication; keep unresolved owner decisions as gates rather than silently choosing a value.

**Focused verification gate**: Run the release/CI dry-run and its negative gate cases from a clean checkout; assert the required pin, branch/event guard, artifact, and publication precondition.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 25. Thread `PRRT_kwDOTNkHgM6QDfwS` — Resolve the empty codec test contradiction

- File: `docs/issues/14-translation-pipeline.md`
- Line: 55
- Finding summary: `Resolve the empty codec test contradiction`

**Normative resolution**: Revise the referenced contract so it normatively resolves “Resolve the empty codec test contradiction” and does not leave the reported ambiguity or failure mode to implementation choice.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 26. Thread `PRRT_kwDOTNkHgM6QDfwT` — Include fallbackTitle in the extractor API

- File: `docs/issues/11-content-extractor.md`
- Line: 17
- Finding summary: `Include fallbackTitle in the extractor API`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Include fallbackTitle in the extractor API”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 27. Thread `PRRT_kwDOTNkHgM6QDfwU` — Distinguish VOICEVOX port squatting in availability

- File: `docs/issues/17-voicevox-provider.md`
- Line: 18
- Finding summary: `Distinguish VOICEVOX port squatting in availability`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Distinguish VOICEVOX port squatting in availability”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.

## 28. Thread `PRRT_kwDOTNkHgM6QDfwX` — Avoid saying the queue is empty after all articles fail

- File: `docs/issues/25-notifications-run-log.md`
- Line: 25
- Finding summary: `Avoid saying the queue is empty after all articles fail`

**Normative resolution**: Revise the referenced contract so it normatively enforces “Avoid saying the queue is empty after all articles fail”, including the affected positive path, the named negative/boundary path, and compatibility with the linked canonical sections.

**Focused verification gate**: Run a focused check for the named behavior with a normal case, the finding's boundary/negative case, and a regression case; assert the documented result and cross-reference consistency.

**Completion boundary**: this section records a design/acceptance contract for later implementation and repository full-validation gates. It does not claim implementation, test, build, CI, security, or release validation is complete.
