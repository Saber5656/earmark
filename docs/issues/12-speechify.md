# Speechify: HTML → speakable paragraphs

## Summary

Implement `src/content/speechify.ts`: the deterministic 13-rule transformation from extracted `contentHtml` to `SpeakableDoc` (plain-text paragraphs safe for TTS, translation, logs, and DB), plus the shared sentence splitter and the reading-time estimator.

## Context

Full-text reading is the v1 product (P5); listenability depends on these rules (DESIGN §8.3). This module is also the sanitization choke point for control/bidi characters (boundary B8) — everything downstream assumes its output is clean plain text. The sentence splitter feeds translation chunking (issue 14) and utterance sizing (issue 16); the reading-time estimate (§8.6) feeds chapter intros and dry-run plans.

## Scope

- `src/content/speechify.ts` (`speechify`, `estimateMinutes`, `countWords`), `src/core/textseg.ts` (`splitSentences` — lives in `core/` so `tts/` and `translate/` may import it without violating DESIGN §2.2 layering), i18n placeholder strings added to `src/core/i18n.ts` (create the file with just these entries; issue 21 extends it), tests.

## Detailed Requirements

1. API: `speechify({contentHtml, placeholderLang: 'ja'|'en'}) → SpeakableDoc` where `SpeakableDoc = { paragraphs: string[] }`. `placeholderLang` is the configured output language (DESIGN §8.3 note: placeholders are emitted in output language; if the body is later translated, translators keep/carry them).
2. Implementation approach: parse `contentHtml` with jsdom, walk the body applying the DESIGN §8.3 rule table in order. Encode each rule exactly:
   - R1 every `<pre>` and every block-level `<code>` (parent is not a phrasing context: `p/li/a/span/em/strong/td/h1-h6`) → one placeholder paragraph `（コード例は省略）` / `(code sample omitted)` **regardless of length**; consecutive placeholder paragraphs collapse to one.
   - R2 inline `<code>` (phrasing context) with text ≤ 40 chars → literal text inline; longer inline `<code>` → inline placeholder `（コード）` / `(code)`.
   - R3 `<table>` → placeholder paragraph `（表は省略）` / `(table omitted)`.
   - R4 `<img>/<figure>/<svg>/<video>/<iframe>` → if `alt` or `<figcaption>` text present: paragraph `図: <text>` / `Figure: <text>`; else drop. Nested figure+img emits once.
   - R5 `<a>` → anchor text; anchor text that itself matches `^https?://` → hostname only.
   - R6 bare URLs in text nodes (`https?://\S+`) → hostname.
   - R7 `<h1>-<h6>` → own paragraph.
   - R8 `<li>` → one paragraph per item (nested lists flatten depth-first; ordered/unordered treated alike).
   - R9 `<blockquote>` → its first paragraph prefixed `引用: ` / `Quote: ` (subsequent paragraphs unprefixed).
   - R10 footnote markers: strip standalone `[n]`/`[note]`-style bracketed digits (`\[\d{1,3}\]`) and `†`/`‡`.
   - R11 strip: C0 controls except `\n` (then `\n` inside a paragraph → space), C1, zero-width U+200B–U+200D & U+FEFF, bidi controls U+202A–U+202E & U+2066–U+2069. (ANSI escapes die with their ESC via C0.)
   - R12 collapse whitespace runs to single space (ja text: also remove spaces between two CJK chars introduced by collapsing); trim; drop empty paragraphs.
   - R13 emoji kept.
   - Paragraph boundaries: block-level elements (`p, h1-6, li, blockquote children, pre/table placeholders, figure captions, br+br`) delimit paragraphs.
3. Placeholder strings live in `core/i18n.ts` under keys `speechify.codeOmitted|tableOmitted|figure|quotePrefix` with `ja`/`en` variants exactly as written above.
4. `splitSentences(text, lang: 'ja'|'en'|string): string[]` (in `src/core/textseg.ts`) — `Intl.Segmenter(langTag, {granularity: 'sentence'})` with mapping: known 639-1 code passed through, `und`/unknown → `'en'`. Segments are trimmed and empties dropped; consumers must NOT reconstruct the original by concatenation (inter-sentence whitespace is not preserved) — consumers merge with their own separator rules (issues 14/16).
5. `estimateMinutes(text, lang)`: `lang === 'ja'` → `ceil(chars/400)`; **all other values including `'und'`** → `ceil(words/160)` where words = whitespace-split count; minimum 1. (DESIGN §8.6.)
6. `countWords(text, lang)` used for `articles.word_count`: ja → code-point count of non-whitespace chars; all other langs → whitespace-split word count (document the semantic in code; aligned with `estimateMinutes` denominators).
7. Determinism and purity: no config, no clock, no randomness; property test (same input twice → identical output).

## Acceptance Criteria

- [ ] One fixture per rule R1–R13 passes with exact expected paragraph arrays (table-driven, both placeholder languages).
- [ ] Composite fixture: `test/fixtures/extractor/ja-blog.content.html` (committed by issue 11) passed as `contentHtml` — snapshot matches and contains no `<` characters, no control chars (regex assert), no double spaces.
- [ ] Hostile fixture with ANSI `\x1b[31m`, bidi `‮`, zero-width joins → all stripped (byte-level assert).
- [ ] `splitSentences` cases: ja 「今日は晴れです。明日は雨。」→ 2; en with abbreviations "Dr. Smith went home. He slept." → 2 (accept Intl.Segmenter behavior as ground truth — snapshot, don't fight it); empty string → []; no concatenation-roundtrip assertion (explicitly not guaranteed).
- [ ] `estimateMinutes('あ'.repeat(1200),'ja') === 3`; `estimateMinutes(<320 words>,'en') === 2`; `'und'` uses the word rule; minimum 1 for tiny text.
- [ ] `countWords` matrix: ja text (non-whitespace code points), en text (word count), `und` (word count), empty and whitespace-only input → 0.
- [ ] Purity property test passes.

## Validation

`vitest` table-driven suite (this is the most test-heavy module; aim for exhaustive rule coverage — it is also part of issue 29's security assertions). No manual steps.

## Dependencies

04 (`core/text.ts` sanitizer reused for R11), 11 (input contract + committed extractor fixture).

## Non-goals

Language detection (13); translation (14/15); utterance sizing for TTS (16 — different limits); number/unit verbalization tuning (rely on TTS engines; revisit post-v1 with real listening feedback); markdown input support.

## Design References

DESIGN §8.3 (rule table), §8.6, §13.1 B8, §13.8; issues 14/16 (consumers of `splitSentences`).
