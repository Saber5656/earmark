# Content extractor (defuddle + readability fallback)

## Summary

Implement `src/content/extract.ts`: parse fetched HTML with jsdom (scripts disabled), extract the main article with `defuddle`, fall back to `@mozilla/readability`, and return the `ExtractedArticle` contract (title, metadata, content HTML, lang hint) or `EXTRACT_EMPTY`.

## Context

Boundary B2 (DESIGN §13.1): hostile HTML must be parsed inert. Extraction quality decides digest quality; the two-extractor strategy is fixed in DESIGN §8.2. Downstream, issue 12 consumes `contentHtml` and issue 13 uses `langHint`.

## Scope

- `src/content/extract.ts`, `src/content/types.ts` (ExtractedArticle), fixtures under `test/fixtures/html/`, tests. Runtime deps added: `defuddle@^0.19`, `@mozilla/readability@^0.6`, `jsdom@^29`.

## Detailed Requirements

1. API: `extractArticle({html, url}) → ExtractedArticle`; throws `EarmarkError EXTRACT_EMPTY` when both extractors fail the minimum-content check.
2. jsdom construction: `new JSDOM(html, { url })` — MUST NOT set `runScripts`, MUST NOT set `resources`; pass a `VirtualConsole` that swallows jsdom noise into `logger.debug`. Wrap construction in try/catch → parse failure maps to `EXTRACT_EMPTY` with cause.
3. Primary: `defuddle` — follow the package's documented Node usage for the installed version (constructor with the jsdom document/window + options `{url}`; verify exact API against the pinned version's README at implementation time and encode it in tests). Collect: `content` (HTML string), `title`, `author` → `byline`, `site` → `siteName`, `published` → `publishedAt` (keep raw string), plus `<html lang>` attribute read directly from the DOM → `langHint` (primary subtag lowercased, e.g. `ja` from `ja-JP`; absent/empty → undefined).
4. Minimum-content check: strip tags from candidate `content` (use a fresh jsdom `textContent`), collapse whitespace; length < 200 chars → candidate fails.
5. Fallback: on defuddle throw or failed check, run `new Readability(freshDocument).parse()` on a **fresh** jsdom instance (Readability mutates the DOM); map `content/title/byline/siteName/publishedTime`. Apply the same minimum-content check. Both fail → `EXTRACT_EMPTY` (message says which stages were tried; this is the paywall/JS-page signal surfaced to the user per DESIGN §8.2).
6. Title precedence (DESIGN §8.2): extracted title (non-empty, trimmed, `stripControl`) → else caller-provided capture title (parameter `fallbackTitle?`) → else hostname of `url`. Export as part of the result; ≤ 500 chars enforced by truncation with ellipsis.
7. Result `ExtractedArticle = { title, byline?, siteName?, publishedAt?, langHint?, contentHtml, textLength, extractor: 'defuddle'|'readability' }`.
8. Determinism: no network, no clock, pure function of inputs (verified by calling twice and deep-equal).
9. Fixtures (synthetic, hand-written for this repo — no copied third-party content, DESIGN §16): `ja-blog.html` (headings/paragraphs/code/table/img, `lang="ja"`), `en-blog.html`, `minimal.html` (bare `<p>` ×3), `paywall-stub.html` (nav + teaser < 200 chars), `broken.html` (unclosed tags, nested garbage), `defuddle-hostile.html` (structure chosen so defuddle returns empty → readability succeeds; craft by trial during implementation and freeze).

## Acceptance Criteria

- [ ] All fixtures produce the expected `extractor`, `title`, `langHint`, and non-empty `contentHtml` (snapshot the text form, not raw HTML, to keep snapshots stable).
- [ ] `paywall-stub.html` → `EXTRACT_EMPTY`; `broken.html` does not throw uncaught (either extracts or clean `EXTRACT_EMPTY`).
- [ ] A fixture containing `<script>window.x=1</script>` proves scripts never execute (no side channel: assert `dom.window.x === undefined` in a direct jsdom probe test replicating extractor settings).
- [ ] Title precedence matrix covered (extracted / fallbackTitle / hostname) incl. control-char stripping and 501-char truncation.
- [ ] `lang="ja-JP"` → `langHint: 'ja'`; missing lang → undefined.
- [ ] Repeated call determinism test passes.

## Validation

`vitest` fixture suite; no manual steps required beyond PR review of fixture realism. During implementation, verify the pinned defuddle version's exact constructor/parse API and record it in a code comment with the version number (U-item hygiene).

## Dependencies

01 (fixtures/tooling). (Fetcher not required: input is an HTML string.)

## Non-goals

Speechify transforms (issue 12); charset handling (done in issue 10); readability tuning options; image/asset downloading; markdown conversion (contentHtml stays HTML for issue 12's DOM walk).

## Design References

DESIGN §8.2, §13.1 B2, §16 (fixture policy); ADR-002 (extraction ecosystem rationale).
