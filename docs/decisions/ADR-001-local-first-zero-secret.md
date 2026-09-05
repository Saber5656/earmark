# ADR-001: Fully local, zero-secret architecture

- Status: Accepted (2026-07-11)
- Deciders: product owner (confirmed via requirements interview), Fable (design)

## Context

earmark converts read-later articles into a daily TTS digest. The owner chose: local TTS, local file delivery, execution on their own Mac via launchd, and self-owned capture. The project will be published as OSS, so defaults must be safe for arbitrary users.

A tension appeared late in requirements: articles in any language must be **translated** into the configured output language (ja/en), which normally pushes toward cloud APIs (DeepL, cloud LLMs).

## Decision

1. v1 runs **entirely on the user's machine**. Permitted network egress: fetching the user's own article URLs, calls to loopback services (VOICEVOX, Ollama), and one-time local model downloads (Kokoro ONNX, Ollama models pulled by the user).
2. v1 has **zero secrets**: no API keys, tokens, or accounts anywhere (config, DB, env). Translation uses a **local LLM via Ollama** (default model `gemma3:12b`) behind a provider interface (ADR-004).
3. No listening sockets, no telemetry, no auto-update, user-scope LaunchAgent only (no daemons, no elevated privileges).

## Consequences

- Setup requires installing VOICEVOX, Ollama (+ model pull), and ffmpeg → mitigated by `earmark doctor` and SETUP docs (issues 27, 30).
- Local translation is slower and lower quality than DeepL/cloud LLMs → accepted; quality/latency measured via run timings (DESIGN §17); cloud providers remain possible as v2 plugins **but must not reintroduce secrets into repo/DB defaults** (standing constraint).
- Privacy: article text never leaves the machine (unless the user points provider base URLs off-loopback, which triggers a warning — DESIGN §13.6).
- Release hygiene stays simple: nothing secret to leak in the repo or in support logs.

## Alternatives considered

- **DeepL API free tier** for translation: best ja↔en quality, but introduces an API key, an account, per-user quota, and cloud egress of article text — rejected for v1 defaults.
- **Cloud LLM (OpenAI/Gemini)**: same secret/egress objections, plus cost — rejected for v1 defaults.
- **Dedicated local MT (NLLB-200/Argos)**: no strong maintained Node path and mediocre ja quality (see docs/research/local-translation.md) — rejected.
- **No translation in v1**: contradicts confirmed requirement P7 — rejected.
