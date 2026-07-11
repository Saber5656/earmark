# Translation interface, need-decision, chunker

## Summary

Implement `src/translate/types.ts` (the `TranslationProvider` interface + shared `ProviderHealth` type in `src/core/provider.ts`) and `src/translate/pipeline.ts`: the translate-or-skip decision, paragraph-preserving chunker, numbered-block wire codec, document-level orchestration with progress, and a deterministic `PassthroughProvider` for tests.

## Context

P7 requires translating any-language articles into the output language. ADR-004 fixes the provider abstraction; the chunking/codec logic is provider-independent and lives here so issue 15 (Ollama) stays a thin client. The numbered-block format rationale is in `docs/research/local-translation-selection.md`.

## Scope

- `src/core/provider.ts`, `src/translate/types.ts`, `src/translate/pipeline.ts`, tests. No new runtime deps.

## Detailed Requirements

1. `core/provider.ts`: `export type ProviderHealth = { ok: boolean; detail: string }` (shared by translation and TTS providers; doctor consumes).
2. `translate/types.ts`: the interface exactly as DESIGN §8.5 (`id`, `checkAvailability()`, `translate({paragraphs, sourceLang, targetLang, onProgress})` returning `{paragraphs}` **with identical paragraph count**).
3. `needsTranslation(detectedLang, outputLanguage) → boolean`: false iff equal; `'und'` → true (DESIGN §8.4).
4. Block codec:
   - `encodeBlocks(paragraphs: string[], startIndex: number) → string`: each paragraph rendered as `[[n]]\n<text>\n` with n = startIndex.. (global paragraph numbering across chunks so the model never sees duplicate indices)
   - `decodeBlocks(output: string, expected: {from, to}) → string[]`: parse lines matching `^\[\[(\d+)\]\]$` as delimiters; requires exactly the expected index sequence in order, each block non-empty after trim; violations throw `BlockFormatError` (typed, caught by callers for retry).
5. Chunker `packChunks(paragraphs, chunkChars) → Chunk[]` where `Chunk = { fromIndex, toIndex, paragraphs }`:
   - greedy: append whole paragraphs while `sum(lengths) + markers ≤ chunkChars`
   - single paragraph longer than `chunkChars` → split at sentence boundaries (issue 12 `splitSentences`) into pseudo-paragraphs that are **rejoined with a space** after translation into the original single paragraph slot (mapping tracked in the chunk model: `splitOf?: index`)
   - never split inside a sentence; a single sentence > chunkChars is sent alone as an oversized chunk (log warn).
6. `translateDocument({provider, title, paragraphs, sourceLang, targetLang, chunkChars, onProgress}) → {title, paragraphs}`:
   - skip path: caller checks `needsTranslation` first (function asserts source ≠ target for clarity)
   - title translated as its own single-block chunk (index 0), body paragraphs numbered from 1
   - chunks translated sequentially via `provider.translate` on the chunk's paragraphs; progress callback `(doneChunks, totalChunks)`
   - reassembly: paragraph count must equal input count after split-rejoin; mismatch → throw `EarmarkError TRANSLATE_INVALID_OUTPUT` (retry handling is per-chunk inside the provider or here? — decision: **retry lives in the provider** (it knows transport vs format errors); pipeline throws on structural failure after provider returns).
7. `PassthroughProvider` (test util in `src/translate/passthrough.ts`, shipped in src for e2e reuse): returns paragraphs unchanged but wrapped `«tr:targetLang»` prefix on the first paragraph — deterministic, lets e2e assert the path executed; `checkAvailability` always ok. Not selectable via config schema (constructed directly in tests only).
8. Everything pure/deterministic; no clock, no network.

## Acceptance Criteria

- [ ] Codec roundtrip property: arbitrary paragraph arrays (incl. text containing `[[3]]`-lookalike lines inside a paragraph — parser only honors delimiter lines exactly matching the pattern **and** the expected next index, otherwise treats as content; document this rule) encode→decode losslessly.
- [ ] `decodeBlocks` failure cases: missing index, out-of-order, empty block, extra trailing block → `BlockFormatError`.
- [ ] Chunker: 100 paragraphs of 100 chars with chunkChars 2500 → chunks of ≤ 2500 incl. marker overhead; 6000-char single paragraph splits at sentence bounds and rejoins to exactly one output paragraph.
- [ ] `translateDocument` with PassthroughProvider preserves count/order and calls progress `totalChunks` times.
- [ ] `needsTranslation('und','ja') === true`, `('ja','ja') === false`, `('en','ja') === true`.

## Validation

`vitest` incl. the property test (fast-check optional — if adding it as devDependency, justify in PR per dependency policy; hand-rolled randomized loop with fixed seed is acceptable). No manual steps.

## Dependencies

02 (config types), 13 (lang codes semantics), 12 (`splitSentences`).

## Non-goals

Ollama client (15); caching; parallel chunk translation (sequential by design, DESIGN §17); partial-document salvage on failure (article fails whole, §12.1).

## Design References

DESIGN §8.5; research/local-translation-selection.md (wire format); ADR-004; §12.2 (`TRANSLATE_INVALID_OUTPUT`).
