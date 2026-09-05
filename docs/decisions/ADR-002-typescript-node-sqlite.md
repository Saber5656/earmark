# ADR-002: TypeScript/Node.js stack with better-sqlite3 state store

- Status: Accepted (2026-07-11)
- Deciders: product owner (stack choice), Fable (storage detail)

## Context

Owner selected TypeScript/Node.js (familiarity; extraction ecosystem: defuddle, @mozilla/readability; VOICEVOX/Ollama are HTTP so language-agnostic). The queue needs durable state with transitions (`queued → digested/failed/archived`), dedupe by unique key, and joins (digests ↔ articles), surviving crashes mid-run.

## Decision

1. Node.js ≥ 22 (dev machine runs 26), TypeScript strict, ESM, single npm package with `earmark` bin.
2. State store: **SQLite via `better-sqlite3@12`** (synchronous API, WAL, transactions), one DB file under `~/.local/share/earmark/`.
3. Article text artifacts (extracted/translated) stored in DB TEXT columns (tens of KB each) — no artifact file tree in v1.
4. Directory conventions: XDG-style (`~/.config`, `~/.local/share`, `~/.local/state`, `~/.cache`) — CLI-tool convention, dotfiles-friendly, despite macOS-only support.

## Consequences

- `better-sqlite3` is a native module → prebuilt binaries for macOS arm64/x64 keep install smooth; version pinned exactly (DESIGN §13.7). Known cost: major Node upgrades may need a package update.
- Synchronous DB API fits the sequential batch pipeline; no ORM (hand-written repositories with parameterized SQL only).
- DB columns for article text keep writes atomic with status changes; if sizes ever matter, a v2 migration can externalize files.

## Alternatives considered

- **`node:sqlite` (built-in)**: still marked experimental across current release lines; API churn risk for an OSS tool — rejected for v1, revisit when stable.
- **JSON file store**: zero native deps, but hand-rolled atomicity/indexing/joins; state machine + history queries fit SQL better — rejected.
- **Python stack**: best extraction library (trafilatura), but owner chose TS; defuddle+readability are adequate — rejected.
- **Go single binary**: best distribution, weakest extraction/ja-text ecosystem — rejected.
