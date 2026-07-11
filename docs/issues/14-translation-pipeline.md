# Translation interface, need-decision, chunker

## Summary

Implement `src/core/provider.ts` (shared `ProviderHealth`), `src/translate/types.ts` (the `TranslationProvider` interface), and `src/translate/pipeline.ts`: the translate-or-skip decision, a paragraph-preserving chunker with an explicit original-index mapping, the chunk-local numbered-block wire codec, document-level orchestration with progress, and a deterministic `PassthroughProvider` for tests.

## Context

P7 requires translating any-language articles into the output language. ADR-004 fixes the provider abstraction; the chunking/codec logic is provider-independent and lives here so issue 15 (Ollama) stays a thin client. The numbered-block format rationale is in `docs/research/local-translation-selection.md`. Block numbering is **chunk-local** (each provider call sees blocks `1..n`); requests are independent LLM contexts, so global numbering buys nothing and would leak pipeline state into the provider interface.

## Scope

- `src/core/provider.ts`, `src/translate/types.ts`, `src/translate/pipeline.ts`, `src/translate/passthrough.ts`, tests. No new runtime deps.

## Detailed Requirements

1. `core/provider.ts`: `export type ProviderHealth = { ok: boolean; detail: string }` (shared by translation and TTS providers; doctor consumes).
2. `translate/types.ts`:
   ```ts
   export interface TranslationProvider {
     readonly id: string;                                  // "ollama"
     checkAvailability(): Promise<ProviderHealth>;
     translate(req: {
       paragraphs: string[];                               // ONE chunk's paragraphs; single-line strings (speechify output contract)
       sourceLang: string;                                 // ISO 639-1 or 'und'
       targetLang: 'ja' | 'en';
     }): Promise<{ paragraphs: string[] }>;                // same count & order, chunk-local
   }
   ```
   (No progress callback on the provider — progress is a `translateDocument` concern.)
3. `needsTranslation(detectedLang, outputLanguage) → boolean`: false iff equal; `'und'` → true (DESIGN §8.4).
4. Block codec (chunk-local; used by providers to build/parse the wire text):
   - `encodeBlocks(paragraphs: string[]) → string`: block `i` (1-based) rendered as `[[i]]\n<text>\n`. Precondition: each paragraph is single-line (speechify guarantees no `\n`); assert and throw on violation. If a paragraph's entire text itself matches `^\[\[\d+\]\]$`, prefix one space at encode time (decode trims, so content is preserved semantically).
   - `decodeBlocks(output: string, expectedCount: number) → string[]`: delimiter lines are lines exactly matching `^\[\[(\d+)\]\]$`; indices must be exactly `1..expectedCount` in order; each block joined from its content lines with a single space, trimmed, must be non-empty; violations throw `BlockFormatError` (typed; providers catch for retry).
5. Chunker `packChunks(paragraphs: string[], chunkChars: number) → {chunks: Chunk[], oversized: boolean}` where
   ```ts
   type Chunk = { items: Array<{ originalIndex: number; text: string }> };  // originalIndex: 0-based body-paragraph index
   ```
   - greedy: append whole paragraphs while `sum(item lengths) + marker overhead ≤ chunkChars`
   - a single paragraph longer than `chunkChars` is split at sentence boundaries (`splitSentences` from `core/textseg.ts`, issue 12) into multiple items sharing the same `originalIndex`
   - a single sentence > chunkChars is emitted alone as an oversized item; `oversized: true` in the result (caller logs; this function stays pure — no logger)
   - never split inside a sentence.
6. `translateDocument({provider, title, paragraphs, sourceLang, targetLang, chunkChars, onProgress}) → Promise<{title, paragraphs}>`:
   - asserts `sourceLang !== targetLang` (callers gate with `needsTranslation`)
   - title translated first as its own single-paragraph chunk
   - body chunks from `packChunks`, each translated sequentially via `provider.translate({paragraphs: chunk items' texts, ...})`
   - `onProgress(doneChunks, totalChunks)` called by `translateDocument` exactly once after each completed chunk (title chunk included in the counts)
   - reassembly: group returned texts by `originalIndex`, join multi-item groups with a single space, order by index; output paragraph count must equal input count → else throw `EarmarkError TRANSLATE_INVALID_OUTPUT` (structural failure after the provider already succeeded transport-wise).
7. `PassthroughProvider` (`src/translate/passthrough.ts`, shipped in src for e2e reuse): returns each paragraph as `⟪tr:<targetLang>⟫` + original text — deterministic, count-preserving; `checkAvailability` always ok. Not selectable via config (constructed directly by tests).
8. Everything pure/deterministic; no clock, no network, no logging.

## Acceptance Criteria

- [ ] Codec roundtrip property: randomized single-line paragraph arrays (seeded loop ≥ 200 cases, lengths 0–4 paragraphs × 0–3000 chars, including paragraphs that are exactly `[[3]]` and paragraphs containing `[[2]]` mid-text) encode→decode losslessly modulo the documented trim/space-prefix rule.
- [ ] `encodeBlocks` throws on a paragraph containing `\n`.
- [ ] `decodeBlocks` failure table: missing index, out-of-order, duplicate index, empty block, extra trailing block, zero delimiters → `BlockFormatError`.
- [ ] Chunker: 100 × 100-char paragraphs at chunkChars 2500 → every chunk ≤ 2500 incl. marker overhead; a 6,000-char paragraph splits at sentence bounds into items sharing one `originalIndex` and reassembles to exactly one output paragraph; a 6,000-char single sentence → `oversized: true`.
- [ ] `translateDocument` with PassthroughProvider: count/order preserved; every output paragraph carries the `⟪tr:ja⟫` prefix; `onProgress` called exactly `totalChunks` times with monotonically increasing `done`.
- [ ] `needsTranslation('und','ja') === true`, `('ja','ja') === false`, `('en','ja') === true`.

## Validation

`vitest` incl. the seeded randomized codec test (hand-rolled loop with a fixed seed; no new dev dependency). No manual steps.

## Dependencies

02 (config types), 12 (`splitSentences` in `core/textseg.ts`), 13 (lang code semantics).

## Non-goals

Ollama client (15); caching; parallel chunk translation (sequential by design, DESIGN §17); partial-document salvage on failure (article fails whole, §12.1).

## Design References

DESIGN §8.5 (interface, chunk-local numbering), §12.2 (`TRANSLATE_INVALID_OUTPUT`); research/local-translation-selection.md (wire format); ADR-004.
