# Hardened HTTP fetcher

## Summary

Implement `src/content/fetch.ts`: fetch an article URL with undici under strict guards — timeouts, size cap, redirect re-validation, private-network egress blocking with DNS pinning (best effort), content-type allowlist, and charset-aware decoding. Returns HTML text + final URL or a typed failure.

## Context

This is trust boundary B1 (DESIGN §13.1): the only place earmark talks to the open internet. All limits come from `config.network` (DESIGN §4.1); behavior is specified in §8.1 and §13.3. Failures are article-scoped and feed the retry policy (§12).

## Scope

- `src/content/fetch.ts`, tests (undici MockAgent + local ephemeral `node:http` servers). Runtime dep added: `undici@^8`.

## Detailed Requirements

1. API: `fetchArticleHtml(url: string, net: NetworkConfig, deps?: {lookup?}) → Promise<{finalUrl, html, contentTypeHeader}>`; throws `EarmarkError` with codes `FETCH_TIMEOUT | FETCH_HTTP_<status> | FETCH_TOO_LARGE | FETCH_UNSUPPORTED_TYPE | FETCH_PRIVATE_BLOCKED | FETCH_TOO_MANY_REDIRECTS | FETCH_NETWORK` (map socket/DNS errors to `FETCH_NETWORK` with cause message).
2. Request setup (per attempt): undici `Client`/`request` with `method: GET`, headers `user-agent: net.userAgent`, `accept: text/html,application/xhtml+xml;q=0.9,*/*;q=0.1`, `accept-language: ja,en;q=0.8`; `maxRedirections: 0` (manual loop); timeouts: `headersTimeout` and `bodyTimeout` = `net.timeoutMs`, plus an overall deadline of `net.timeoutMs` × 2 across all redirect hops enforced with `AbortSignal.timeout`.
3. Private-network egress policy (skip entirely when `net.allowPrivateNetworks`):
   - hostname is an IP literal → check with issue-05 classifiers; blocked → `FETCH_PRIVATE_BLOCKED`
   - hostname is a name → `isPrivateHostname` check first; then resolve via injected `lookup` (default `dns.promises.lookup` with `{all: true}`); if **any** resolved address is private → blocked (fail closed); else pin: connect to the first non-private resolved address by using an undici Agent `connect.lookup` callback that returns only vetted addresses (this both validates and pins, mitigating rebinding best-effort per DESIGN §13.3)
   - Re-run the full policy on **every redirect hop**.
4. Redirect loop: statuses 301/302/303/307/308 with a `location` header → resolve relative to current URL → re-validate scheme (http/https only) + policy → count against `net.maxRedirects` (exceed → `FETCH_TOO_MANY_REDIRECTS`). 303 → subsequent GET (already GET). Any other 3xx → treat as `FETCH_HTTP_<status>`.
5. Response acceptance: final status must be 200 (else `FETCH_HTTP_<status>`); `content-type` media type must be `text/html` or `application/xhtml+xml` (parameters ignored, case-insensitive; missing content-type → treat as `text/html` but log warn) else `FETCH_UNSUPPORTED_TYPE`.
6. Body streaming: accumulate chunks; on exceeding `net.maxResponseBytes` destroy the stream and throw `FETCH_TOO_LARGE` (must not buffer beyond cap + one chunk).
7. Charset decoding order: `charset=` parameter of content-type → else scan first 1024 bytes for `<meta charset>` / `<meta http-equiv="content-type">` (ASCII scan on raw bytes) → else UTF-8. Decode with `TextDecoder(label, {fatal: false})`; unknown label → UTF-8 with warn. If decoded text contains U+FFFD above 5% of length, log warn `charset_suspect` (do not fail). `iconv-lite` must NOT be added in this issue (DESIGN §8.1 defers it until a real fixture requires it — record U7 evidence if encountered).
8. The function performs no retries (retry semantics belong to the run-level policy, §12.1 — one fetch attempt per run).
9. All log lines include the target host but never the full HTML (debug may include first 200 chars).

## Acceptance Criteria

- [ ] MockAgent tests: 200 html ok; 404 → `FETCH_HTTP_404`; content-type `application/pdf` → `FETCH_UNSUPPORTED_TYPE`; missing content-type → accepted with warn.
- [ ] Real-local-server tests (`node:http` on 127.0.0.1 with `allowPrivateNetworks:true` for the test config): redirect chain of 3 followed with final URL reported; 6 redirects with `maxRedirects:5` → `FETCH_TOO_MANY_REDIRECTS`; redirect to `http://169.254.1.1/` under `allowPrivateNetworks:false` → `FETCH_PRIVATE_BLOCKED` (policy active even though origin allowed via injected lookup vetting 127.0.0.1 for the test origin — demonstrate with injected lookup); slow-loris body exceeding timeout → `FETCH_TIMEOUT`; 20 MB stream with 10 MiB cap → `FETCH_TOO_LARGE` and connection destroyed early (assert bytes received < cap + 64 KiB).
- [ ] Charset tests: `content-type: text/html; charset=euc-jp` body decodes via TextDecoder('euc-jp'); meta-charset shift_jis page decodes; garbage label falls back UTF-8 with warn.
- [ ] Private-IP literal `http://192.168.1.1/x` and hostname resolving to 10.0.0.5 (injected lookup) both blocked; same inputs pass with `allowPrivateNetworks:true`.
- [ ] No test performs real external network I/O (enforce with undici `MockAgent` global dispatcher + assertion, local listeners exempt).

## Validation

`vitest` suite as above (this issue's tests are reused by issue 29's suite). Manual: one real fetch of a public article on the dev Mac via a scratch script, transcript in PR.

## Dependencies

02, 05.

## Non-goals

Caching/ETags; cookies; JS rendering; proxy support; parallel fetching (orchestrator is sequential); retry loops (issue 23/24 policy).

## Design References

DESIGN §8.1, §13.3, §13.8, §12.1–12.2; issue 05 (classifiers).
