# Digest script builder (verbatim generator + i18n templates)

## Summary

Implement `src/script/types.ts` (the `ScriptGenerator` interface and `DigestScript` model), `src/script/verbatim.ts` (the v1 generator), and the full narration template set in `src/core/i18n.ts` for `ja` and `en`, exactly per DESIGN §9.1–9.2.

## Context

This produces the complete text of the morning episode: opening with date + rundown, per-article intro + full body, closing with failure/queue status. It is ADR-004's designated v2 extension point (a future summary generator swaps in here), so the interface must be honored strictly — pure data-in/data-out, no DB/config access.

## Scope

- `src/script/types.ts`, `src/script/verbatim.ts`, `src/core/i18n.ts` (extend from issue 12), tests with golden snapshots.

## Detailed Requirements

1. Types exactly per DESIGN §9.1: `Segment {kind:'narration'|'body', lang, text}`, `Chapter {title, articleId?, segments}`, `DigestScript {date, chapters}`, and inputs `PreparedArticle {id, title, siteHost, paragraphs, estimatedMinutes}`, `QueueStats {failedThisRun: {id,title}[], permanentlyFailedThisRun: {id,title}[], queueRemaining: number}`, `ScriptGenerator` interface with `generate({date, lang, articles, stats})`.
2. i18n templates (`core/i18n.ts`), keys with exact strings — ja shown, en analog required:
   - `script.openingTitle` = `オープニング` / `Opening`
   - `script.opening` = `earmark デイリーダイジェスト。{date}。今日は{count}本の記事をお届けします。`
   - `script.rundownIntro` = `ラインナップ。` ; `script.rundownItem` = `{n}、『{title}』。`
   - `script.articleIntro` = `{n}本目、『{title}』。{siteHost} より。読了目安、約{minutes}分。`
   - `script.closingTitle` = `クロージング` / `Closing`
   - `script.closingMain` = `以上、{count}本をお届けしました。`
   - `script.closingFailures` = `なお、{count}本の記事は処理に失敗しました。詳細は earmark list で確認できます。`
   - `script.closingPermanent` = `次の記事は3回失敗したため、キューから外れました。{titles}。`
   - `script.closingRemaining` = `キューには残り{count}本の記事があります。` (emitted only when > 0)
   - `script.signOff` = `良い一日を。` / `Have a great day.`
   - en equivalents: natural English, e.g. opening `This is your earmark daily digest for {date}. Today we have {count} article(s).` — singular/plural handled by a minimal `plural(n, one, many)` helper for en; ja ignores plurality.
   - Date rendering: `Intl.DateTimeFormat` — ja `2026年7月11日、金曜日` (era-free, weekday long); en `Friday, July 11, 2026`. Input is the `YYYY-MM-DD` string; construct the date at **local** midnight (date-only semantic; no timezone math beyond that).
3. `VerbatimScriptGenerator.generate` structure (DESIGN §9.2):
   - chapters[0] Opening: one narration segment: opening sentence + (articles ≥ 2 ? rundown intro + items : nothing)
   - one chapter per article, `title` = article title, `articleId` set: first segment narration `articleIntro`, then one `body` segment **per paragraph** (preserving paragraph boundaries for gap insertion downstream — do NOT merge paragraphs)
   - last chapter Closing: narration segments in order: closingMain; closingFailures (only when `failedThisRun.length > 0`, count = its length); closingPermanent (only when non-empty; titles joined `、` / `, `); closingRemaining (only when > 0); signOff.
   - all segments `lang` = input `lang`; all narration text passes `stripControl` defensively.
4. Edge behaviors: `articles.length === 1` → opening says 1本/1 article, no rundown; article title containing `『』` quotes → keep as-is (no escaping; TTS reads them fine); zero articles is a **caller error** (generator throws — orchestrator never calls it with 0, per DESIGN §11.1).
5. Purity: no clock (date passed in), no config (lang passed in), deterministic output.

## Acceptance Criteria

- [ ] Golden snapshots (committed as readable `.txt` renderings, segment-per-line with kind markers) for: ja 3-articles happy path; ja 1-article; ja with 2 failures + 1 permanent + 4 remaining; en 2-articles happy path; en with failures. Reviewer reads the ja goldens for naturalness sign-off.
- [ ] Chapter count = articles + 2; every article chapter's segment count = 1 + paragraphs.length; body paragraphs byte-identical to input.
- [ ] Date rendering exact for 2026-07-11 in both languages (fixed test date).
- [ ] Determinism: two calls deep-equal.
- [ ] No import from `core/db`, `core/config`, or any feature module besides `core/i18n` (lint layering + explicit test).

## Validation

`vitest` snapshot suite; native-speaker (owner) review of ja narration strings happens at PR review — flag any wording change back into this issue file so docs stay canonical.

## Dependencies

02 (i18n file location only), 12 (paragraph semantics), 14 (PreparedArticle provenance).

## Non-goals

Summarization (v2 generator); per-article language mixing (`lang` is uniform in v1 — DESIGN §9.2); SSML or engine-specific markup; utterance sizing (issue 16 consumes this output).

## Design References

DESIGN §9.1, §9.2, §8.6; ADR-004; issue 22 (chapter/paragraph gap consumption).
