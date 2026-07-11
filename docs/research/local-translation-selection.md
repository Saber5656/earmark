# Research: Local translation approach (any language → ja/en)

- Date: 2026-07-11
- Method: comparative assessment from model-ecosystem knowledge (cutoff-safe, all candidates pre-2026); translation quality/latency of the chosen default MUST be re-measured on the dev Mac during issue 15 (acceptance includes a fixture-based quality check).
- Feeds: DESIGN §8.5, ADR-001, ADR-004.

## Requirement

P7: articles captured in any language are translated into the configured output language (`ja` or `en`) before synthesis. ADR-001 requires a local, zero-secret default. Volume: up to 10 articles × ~1,500–6,000 chars each morning; latency budget ≤ 25 min for a fully-cross-language batch (DESIGN §17).

## Candidates

| Approach | Quality (→ja) | Setup | Fit |
|---|---|---|---|
| **Ollama + general instruct LLM** (chosen) | Good with 8–12B multilingual models | `brew install ollama` + `ollama pull` | Local HTTP API, streaming, easy model swap; also the natural host for the v2 summary generator — one runtime serves two future needs |
| NLLB-200 (distilled) | ja quality mediocre; sentence-level MT, weak document cohesion | Python/ONNX pipelines; no maintained Node path | Rejected |
| Argos Translate / OpenNMT | Weak for ja | Python | Rejected |
| Apple on-device Translation framework | Decent | Requires a Swift helper binary + macOS-version coupling | Rejected for v1 (interesting v2 provider) |
| DeepL API free tier | Best ja quality | API key + account + egress | Violates zero-secret default (ADR-001); recorded as v2 cloud provider option |

## Default model choice (config `translation.ollama.model`)

| Model | Size | Notes |
|---|---|---|
| **`gemma3:12b`** (default) | ~8 GB | Strong multilingual incl. ja; dense, no thinking-mode output to sanitize |
| `gemma3:4b` | ~3 GB | Documented low-RAM alternative; noticeably weaker ja prose |
| `qwen3:8b` | ~5 GB | Capable but emits thinking blocks by default → output-contract risk; documented alternative for users who configure it off |

The default is a config value, not an architectural commitment; doctor verifies the configured model exists in `ollama /api/tags` and SETUP documents the pull command.

## Wire-format decision (feeds issue 15)

Paragraph fidelity matters (paragraph gaps drive audio pacing). Chunks are sent as numbered blocks:

```
[[1]]
<paragraph text>
[[2]]
<paragraph text>
```

with the system prompt requiring identical block markers in the output. Parser splits on `^\[\[\d+\]\]$` lines; block-count mismatch or empty block → one retry → article failure `TRANSLATE_INVALID_OUTPUT`. Numbering is chunk-local (each provider request sees blocks 1..n; requests are independent LLM contexts). This is more robust than JSON output (models corrupt JSON escaping in long prose) and than free text (loses paragraph mapping).

## Risks / follow-ups

- Throughput on 16 GB machines with 12B model may exceed budget → measure in issue 15; remediation is documented model downgrade, not architecture change.
- Very long single paragraphs (> chunkChars) are sentence-split before packing (DESIGN §8.5).
- Prompt injection via article text: impact ceiling analysis in DESIGN §13.5.
