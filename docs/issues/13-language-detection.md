# Language detection

## Summary

Implement `src/content/language.ts`: decide each article's language (ISO 639-1 or `und`) from franc trigram detection over the speakable text plus the extractor's `<html lang>` hint, per the DESIGN §8.4 decision procedure.

## Context

The detected language drives the translate-or-not decision (issue 14): wrong detection either wastes a translation pass or ships untranslated audio. The procedure is deliberately conservative with short texts and defers to the page's own `lang` attribute when statistical confidence is low. Accuracy limits are a known unknown (ISSUE_PLAN U6).

## Scope

- `src/content/language.ts`, tests. Runtime dep added: `franc@^6` (ESM).

## Detailed Requirements

1. API: `detectLanguage({paragraphs, langHint}) → {lang: string, source: 'franc'|'hint'|'und'}` — `lang` is ISO 639-1 (2-letter) or `'und'`.
2. Input text: `paragraphs.join('\n')` truncated to the first 4000 characters (DESIGN §8.4).
3. franc usage: call `francAll(text, {minLength: 40})`; result codes are ISO 639-3. Map to 639-1 via an in-repo constant table `ISO_639_3_TO_1` covering at least: jpn→ja, eng→en, cmn→zh, kor→ko, fra→fr, deu→de, spa→es, por→pt, ita→it, rus→ru, nld→nl, pol→pl, tur→tr, vie→vi, tha→th, ind→id, arb/ara→ar, hin→hi, ukr→uk, swe→sv, dan→da, nor/nob→no, fin→fi, ces→cs, ell→el, heb→he (extendable; unmapped → treated as undetermined). No new dependency for mapping.
4. Confidence rule (deterministic):
   - text length < 40 → franc unusable → fall to hint
   - text length ≥ 200 AND franc top result maps to a known 639-1 AND (single result OR topScore/secondScore ≥ 1.05 where scores are francAll's normalized distances — compute ratio guarding divide-by-zero) → **franc wins** (`source:'franc'`)
   - otherwise if `langHint` is a 2-letter code in our mapping's value set → **hint wins** (`source:'hint'`)
   - else if franc produced a mapped top result (40 ≤ len < 200 case) → franc with `source:'franc'`
   - else `{lang:'und', source:'und'}`.
5. Log at debug: `{francTop, ratio, hint, decision}` for tuning U6 later.
6. Pure function; franc import is ESM-only — ensure the package's ESM interop works under NodeNext (smoke-tested by the suite itself).

## Acceptance Criteria

- [ ] Fixture texts (≥ 300 chars each, synthetic): Japanese → `ja/franc`; English → `en/franc`; Chinese (simplified) → `zh/franc`; Korean → `ko/franc`.
- [ ] 30-char Japanese text with `langHint:'ja'` → `ja/hint`; same text without hint → `und`.
- [ ] 100-char English text without hint → `en/franc` (mid-length branch); with `langHint:'fr'` and franc confidently `eng` at ≥200 chars → `en/franc` (franc precedence proven).
- [ ] Text in an unmapped language (e.g. random Basque fixture if franc detects `eus`) → falls to hint/und path, never returns a 3-letter code.
- [ ] Return type never contains uppercase or region subtags.

## Validation

`vitest` fixtures as above. Post-merge tuning note: real-world misdetections during wave-4 testing get recorded as U6 evidence with the debug log line attached.

## Dependencies

12 (speakable paragraphs as input; shares fixtures).

## Non-goals

Mixed-language segment-level detection (whole-article decision only in v1); translation need decision (issue 14); expanding the mapping table beyond listed languages unless a test fixture demands it.

## Design References

DESIGN §8.4; ISSUE_PLAN U6; issue 14 (consumer).
