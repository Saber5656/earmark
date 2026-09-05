# ADR-004: Pluggable provider interfaces for script generation, translation, and TTS

- Status: Accepted (2026-07-11)
- Deciders: product owner (P5 "verbatim now, summary later"; P7 translation; P2 local TTS), Fable (interface design)

## Context

Three pipeline stages are known extension points:
- **Script generation**: v1 reads full text verbatim; owner explicitly wants LLM summarization as v2 without re-architecture.
- **Translation**: v1 uses local Ollama; cloud providers (DeepL, cloud LLMs) are plausible v2 plugins.
- **TTS**: ja uses VOICEVOX, en uses Kokoro with `say` fallback; future voices/engines likely.

## Decision

Define three narrow interfaces (full signatures in DESIGN §8.5, §9.1, §10.1), selected by config, constructed in one registry module each:

| Interface | v1 implementations | Config key |
|---|---|---|
| `ScriptGenerator.generate(input) → DigestScript` | `verbatim` | (fixed in v1; key reserved) |
| `TranslationProvider.translate(paragraphs, from, to)` | `ollama` | `translation.provider` |
| `TtsProvider.synthesize(utterance, outWavPath)` | `voicevox` (ja), `kokoro` (en), `say` (en) | `tts.<lang>.provider` |

Shared design rules:
- Interfaces are **data-in/data-out**; providers never touch the DB, CLI, or each other.
- Every provider exposes `checkAvailability()` consumed by `earmark doctor` and pre-run checks.
- Chunking/utterance-splitting live **outside** providers (shared pipeline code) so provider implementations stay minimal.
- v1 ships exactly the implementations above; config enum values are closed sets (zod), widened per release.

## Consequences

- v2 summarization = one new `ScriptGenerator` + config enum widening; no orchestrator change.
- Slight v1 overhead (three interface files + registries) — accepted as the price of the owner's explicit v2 path.
- Provider-specific setup problems surface uniformly through doctor rather than mid-run.

## Alternatives considered

- **Hard-code v1 implementations, refactor later**: cheaper now, but the owner's requirement P5 explicitly reserves the summary swap, and retrofitting interfaces under a working pipeline is riskier — rejected.
- **Full plugin system (dynamic loading of third-party packages)**: unnecessary surface and a supply-chain risk for v1 — rejected; in-repo providers only.
