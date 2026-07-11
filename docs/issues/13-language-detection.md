# Language detection

## Summary

Implement `src/content/language.ts`: decide each article's language (ISO 639-1 or `und`) from franc trigram detection over the speakable text plus the extractor's `<html lang>` hint, per the DESIGN §8.4 decision procedure.

## Context

The detected language drives the translate-or-not decision (issue 14): wrong detection either wastes a translation pass or ships untranslated audio. The procedure is deliberately conservative with short texts and defers to the page's own `lang` attribute when statistical confidence is low. Accuracy limits are a known unknown (ISSUE_PLAN U6).

## Scope

- `src/content/language.ts`, tests. Runtime dep added: `franc@^6` (ESM).

## Detailed Requirements

1. API: `detectLanguage({paragraphs, langHint}, deps?: {francAll?: FrancAllFn, logger?: Pick<Logger,'debug'>}) → {lang: string, source: 'franc'|'hint'|'und'}` — `lang` is ISO 639-1 (2-letter) or `'und'`. Both deps optional: default `francAll` is the real franc; omitting `logger` keeps the function side-effect-free (the orchestrator passes the real logger).
2. Input text: `paragraphs.join('\n')` truncated to the first 4000 characters (DESIGN §8.4).
3. franc usage: `francAll(text, {minLength: 40})` returns `[code, score]` pairs, **higher score = better match**, codes ISO 639-3. Map to 639-1 via an in-repo constant table `ISO_639_3_TO_1` covering at least: jpn→ja, eng→en, cmn→zh, kor→ko, fra→fr, deu→de, spa→es, por→pt, ita→it, rus→ru, nld→nl, pol→pl, tur→tr, vie→vi, tha→th, ind→id, arb/ara→ar, hin→hi, ukr→uk, swe→sv, dan→da, nor/nob→no, fin→fi, ces→cs, ell→el, heb→he (extendable; unmapped → undetermined). Separate constant `KNOWN_ISO_639_1`: the mapping's value set **plus** any ISO 639-1 codes accepted as hints (initialize = value set; DESIGN §8.4 hint semantics).
4. Confidence rule (deterministic):
   - `ratio = second === undefined || second.score === 0 ? Infinity : top.score / second.score`
   - text length < 40 → franc unusable → fall to hint
   - text length ≥ 200 AND top maps to a known 639-1 AND `ratio ≥ 1.05` → **franc wins** (`source:'franc'`)
   - otherwise if `langHint` ∈ `KNOWN_ISO_639_1` → **hint wins** (`source:'hint'`)
   - else if franc produced a mapped top result (covers 40 ≤ len < 200, and low-ratio-no-hint) → franc (`source:'franc'`)
   - else `{lang:'und', source:'und'}`.
5. Debug log (only when `logger` provided): `{francTop, ratio, hint, decision}` for tuning U6 later.
6. franc is ESM-only — ensure interop under NodeNext (smoke-tested by the suite itself).

## Acceptance Criteria

- [ ] Fixture texts (≥ 300 chars each, synthetic): Japanese → `ja/franc`; English → `en/franc`; Chinese (simplified) → `zh/franc`; Korean → `ko/franc`.
- [ ] 30-char Japanese text with `langHint:'ja'` → `ja/hint`; same text without hint → `und`.
- [ ] 100-char English text without hint → `en/franc` (mid-length branch); with `langHint:'fr'` and franc confidently `eng` at ≥200 chars → `en/franc` (franc precedence proven).
- [ ] Low-confidence long text (injected `francAll` returning `[['eng',1.0],['deu',0.97]]` → ratio < 1.05) with `langHint:'de'` → `de/hint` (the hint-wins branch at ≥200 chars, deterministic via the seam).
- [ ] Truncation: decisive Japanese content placed only after character 4000 of an otherwise-English text does not affect the decision.
- [ ] Text in an unmapped language (injected top `eus`) → falls to hint/und path, never returns a 3-letter code.
- [ ] Return type never contains uppercase or region subtags; no logging occurs when `logger` is omitted (spy on a global sink).

## Validation

`vitest` fixtures as above. Post-merge tuning note: real-world misdetections during wave-4 testing get recorded as U6 evidence with the debug log line attached.

## Dependencies

12 (speakable paragraphs as input; shares fixtures).

## Non-goals

Mixed-language segment-level detection (whole-article decision only in v1); translation need decision (issue 14); expanding the mapping table beyond listed languages unless a test fixture demands it.

## Design References

DESIGN §8.4; ISSUE_PLAN U6; issue 14 (consumer).
