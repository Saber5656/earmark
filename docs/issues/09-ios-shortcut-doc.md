# iOS Shortcut recipe documentation (`docs/SHORTCUT.md`)

## Summary

Write `docs/SHORTCUT.md`: a step-by-step, screenshot-free recipe for building the "Save to earmark" iOS Shortcut that writes the DESIGN §6.1 inbox file from the Share Sheet, plus testing and troubleshooting guidance. Documentation-only issue.

## Context

The Shortcut is the user-built half of the iPhone capture channel (ADR-005). It must be reproducible by a non-expert from text alone (we cannot ship a `.shortcut` binary in-repo for v1), and it must produce files that issue 08 ingests without modification.

## Scope

- `docs/SHORTCUT.md` (English; a short Japanese quick-section at the top is allowed since end-users are Japanese-first — keep both in one file). No code changes.

## Detailed Requirements

1. Prerequisites section: iCloud Drive enabled; the `earmark/inbox` folder exists (instruct to run `earmark ingest` once on the Mac first, which creates it via issue 08, and wait for it to appear in Files).
2. Exact action-by-action recipe (Shortcuts app, iOS 17+ wording):
   1. New Shortcut → rename "Save to earmark" → enable **Show in Share Sheet**; accepted types: **URLs** only.
   2. Action **Get URLs from Input** (input: Shortcut Input) — normalizes share payloads to a URL list.
   3. Action **Repeat with Each** over the URLs (handles multi-URL shares):
      - **Dictionary** action with keys: `v` → Number `1`; `url` → `Repeat Item`; `sharedAt` → **Format Date** of Current Date, format ISO 8601.
      - **Save File** (Files action): service iCloud Drive; destination path `earmark/inbox`; **Ask Where to Save: Off**; **Overwrite: Off**; Subpath/filename: `Current Date` formatted `yyyyMMdd-HHmmss` + `-` + a 4-char random component — document the Shortcuts technique: **Format Date** custom `yyyyMMdd-HHmmss` into a variable, plus **Random Number** between 1000–9999 as the suffix, filename extension `.json`.
   4. Note: passing a Dictionary to Save File stores it as JSON text — state this explicitly and instruct verification of the produced file's content via the Files app (long-press → Quick Look).
3. Reproduce the **full DESIGN §6.1 inbox contract** in the doc, not just an example body: UTF-8 JSON, `v: 1`, required `url`, optional `title` ≤ 500 chars, ≤ 64 KiB file size, `.json` extension (`.txt`-with-URL fallback), filename is informative-only (earmark never trusts or reuses it), invalid files are moved to `inbox/rejected/` with the reason logged. State that `title` is omitted by this recipe (v1 minimal; a title-capturing variant is an optional appendix using **Get Name** on Safari shares, marked "may not work in all share contexts").
4. Testing section: share any article from Safari → run the shortcut → within Files confirm the JSON appears under `earmark/inbox` → on the Mac run `earmark ingest` → confirm `imported 1` and the file moved to `inbox/processed/`.
5. Troubleshooting table: file never appears on Mac (iCloud sync pending → Files app pull-to-refresh, check same Apple ID); `rejected/` contains the file (open it, compare against the contract; most common: shared a non-URL); shortcut asks where to save every time (Ask Where to Save left On); duplicate saves (expected: earmark dedupes by URL); morning digest missed a just-shared URL (sync latency — rolls over to tomorrow, ADR-005 consequence).
6. Keep terminology aligned with the config: if the user changed `paths.inboxDir`, the Files destination must match — and iPhone capture only works when that folder is visible in iCloud Drive/Files on iOS. A non-iCloud local path keeps CLI capture working but this Shortcut cannot target it; say so explicitly. Show how to print the active path (`earmark config get paths.inboxDir` + `earmark config path`).
7. Add a link to this doc from `docs/DESIGN.md` §6.1? No — DESIGN stays stable; instead issue 30's README/SETUP links here (note this dependency for issue 30; no DESIGN edit in this issue).

## Acceptance Criteria

- [ ] A reader can build the Shortcut from the text alone without guessing any action name or option (review by a second person or agent following the steps literally against the Shortcuts action catalog).
- [ ] The documented filename scheme and JSON body match DESIGN §6.1 exactly (field names, `v:1`, extension).
- [ ] Testing + troubleshooting sections present with the items listed above.
- [ ] File is valid Markdown, English main body, optional short Japanese quickstart block at top.

## Validation

Manual, on a real iPhone: build the Shortcut following only the written steps; share one article; run `earmark ingest --json` on the Mac and verify the machine-readable report shows `imported` length 1 (the doc's testing section references this exact check); paste the transcript and the produced JSON file body (URL redacted if desired) into the PR as evidence. Requires issue 08 merged and a real iCloud account (wave-1 gate).

## Dependencies

08 (contract implemented and directory bootstrap available).

## Non-goals

Shipping an importable `.shortcut`/iCloud link (v2 idea — requires publishing and maintaining a signed shortcut); macOS Share Sheet recipe; title extraction guarantees; Android/anything non-iOS.

## Design References

DESIGN §6.1, §13.1 B3; ADR-005; issue 08 (ingest behavior the doc's testing section relies on).
