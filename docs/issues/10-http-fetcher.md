# Hardened HTTP fetcher

## Summary

Implement `src/content/fetch.ts`: fetch an article URL with undici under strict guards — one overall timeout, size cap, redirect re-validation, private-network egress blocking with DNS vetting/pinning (best effort), content-type allowlist, and charset-aware decoding. Returns HTML text + final URL or a typed failure.

## Context

This is trust boundary B1 (DESIGN §13.1): the only place earmark talks to the open internet. All limits come from `config.network` (DESIGN §4.1); behavior is specified in §8.1 and §13.3. Failures are article-scoped and feed the retry policy (§12).

## Scope

- `src/content/fetch.ts`, tests (undici MockAgent + local ephemeral `node:http` servers). Runtime dep added: `undici@^8`.

## Detailed Requirements

1. API:
   ```ts
   fetchArticleHtml(url: string, net: NetworkConfig, deps?: {
     lookup?: LookupFn;                       // DNS seam, default dns.promises.lookup-based
     logger?: Logger;                         // from core/logger (issue 04)
   }) → Promise<{finalUrl: string, html: string, contentTypeHeader: string | null}>
   ```
   Throws `EarmarkError` with codes `FETCH_TIMEOUT | FETCH_HTTP_<status> | FETCH_TOO_LARGE | FETCH_UNSUPPORTED_TYPE | FETCH_PRIVATE_BLOCKED | FETCH_TOO_MANY_REDIRECTS | FETCH_NETWORK` (socket/DNS errors → `FETCH_NETWORK` with cause message).
2. `LookupFn` contract (exact): `(hostname: string) => Promise<Array<{address: string, family: 4|6}>>` returning **all** resolved addresses. Default implementation wraps `dns.promises.lookup(hostname, {all: true, verbatim: true})`.
3. User-Agent: `net.userAgent` with the `{version}` placeholder substituted with the package version (issue 02 stores the template literally).
4. Overall deadline: a single `AbortSignal.timeout(net.timeoutMs)` budget covers **all** redirect hops, headers, and body (DESIGN §8.1: one overall timeout). undici per-request `headersTimeout`/`bodyTimeout` are set to `net.timeoutMs` as inner backstops.
5. Private-network egress policy (skipped entirely when `net.allowPrivateNetworks`), applied to the initial URL and **re-applied on every redirect hop**:
   - scheme must be `http:`/`https:` (redirects to anything else → `FETCH_PRIVATE_BLOCKED`? No — `URL_INVALID` semantics don't apply mid-flight; use `FETCH_NETWORK` with message `unsupported redirect scheme`)
   - hostname checks via issue 05: `isPrivateHostname` → blocked; IP-literal hostname → `isBlockedIp` → blocked
   - name hostnames: resolve via `deps.lookup`; **any** returned address failing `isBlockedIp === false` check (i.e. any private/special address present) → `FETCH_PRIVATE_BLOCKED` (fail closed)
   - pinning: perform the hop's request through a per-hop undici `Agent` whose `connect.lookup` callback returns only the vetted addresses (validates and pins in one mechanism, mitigating rebinding best-effort per DESIGN §13.3); one article fetch may build up to `maxRedirects + 1` short-lived Agents — performance is irrelevant (sequential pipeline), correctness wins. Close each Agent after its hop.
6. Redirect loop: manual (`maxRedirections: 0`); statuses 301/302/303/307/308 with `location` → resolve relative to current URL → policy re-check → count against `net.maxRedirects` (exceed → `FETCH_TOO_MANY_REDIRECTS`). Other 3xx → `FETCH_HTTP_<status>`. Every non-final response body must be drained or destroyed before following the next hop (no socket leaks).
7. Response acceptance: final status must be 200 (else `FETCH_HTTP_<status>`); `content-type` media type must be `text/html` or `application/xhtml+xml` (parameters ignored, case-insensitive). **Missing** content-type header → treated as `text/html` with a `logger.warn` (DESIGN §8.1 documents this exception); any other media type → `FETCH_UNSUPPORTED_TYPE`.
8. Body streaming: accumulate chunks; on exceeding `net.maxResponseBytes` destroy the stream and throw `FETCH_TOO_LARGE` (must not buffer beyond cap + one chunk).
9. Charset decoding order: `charset=` parameter of content-type → else ASCII-scan first 1024 raw bytes for `<meta charset>` / `<meta http-equiv="content-type" ...>` → else UTF-8. Decode via `TextDecoder(label, {fatal: false})`; unknown label → UTF-8 with warn. Decoded text with > 5% U+FFFD → `logger.warn('charset_suspect')` (do not fail). `iconv-lite` must NOT be added in this issue (DESIGN §8.1 defers until a real fixture requires it — record U7 evidence if encountered).
10. No retries inside this function (one fetch attempt per run; retry policy is issues 23/24).
11. Logging: warn/debug only; include target host, never full HTML (debug may carry first 200 chars).

## Acceptance Criteria

- [ ] MockAgent tests: 200 html ok; 404 → `FETCH_HTTP_404`; `application/pdf` → `FETCH_UNSUPPORTED_TYPE`; missing content-type → accepted + warn captured via injected logger.
- [ ] Local-server tests (`node:http` on 127.0.0.1; test config uses `allowPrivateNetworks:true` for loopback reachability except where noted): 3-hop redirect chain followed, `finalUrl` = last URL; 6 redirects with `maxRedirects:5` → `FETCH_TOO_MANY_REDIRECTS`; redirect bodies drained (server asserts connection reuse or test asserts `body.dump()` called via spy).
- [ ] Policy unit tests with `allowPrivateNetworks:false` and injected `lookup` (no real sockets needed): initial hostname resolving to `10.0.0.5` → `FETCH_PRIVATE_BLOCKED`; hostname resolving to `[93.184.216.34, 192.168.0.9]` (mixed) → blocked; redirect hop to `http://169.254.1.1/` → blocked; IP-literal `http://192.168.1.1/x` → blocked without lookup call; same inputs pass with `allowPrivateNetworks:true`.
- [ ] Pinning test: injected lookup returns a vetted address; the per-hop Agent's `connect.lookup` receives and returns exactly that address (spy on the connect option).
- [ ] Timeout: slow body exceeding `timeoutMs` → `FETCH_TIMEOUT`; the budget spans hops (two slow hops that individually fit but jointly exceed → `FETCH_TIMEOUT`).
- [ ] Size cap: 20 MB stream with 10 MiB cap → `FETCH_TOO_LARGE`, bytes received < cap + 64 KiB (server-side counter).
- [ ] Charset: `charset=euc-jp` header decodes; meta-charset shift_jis fixture decodes; garbage label → UTF-8 + warn.
- [ ] No test performs real external network I/O (undici MockAgent global dispatcher with `disableNetConnect()` allowing only 127.0.0.1 listeners).

## Validation

`vitest` suite as above (reused by issue 29). Manual: one real fetch of a public article on the dev Mac via a scratch script, transcript in PR.

## Dependencies

02 (network config, UA template), 04 (logger contract), 05 (`isBlockedIp` / `isPrivateHostname`).

## Non-goals

Caching/ETags; cookies; JS rendering; proxy support; parallel fetching; retry loops (issues 23/24); non-UTF8 legacy encodings beyond TextDecoder's built-ins (U7).

## Design References

DESIGN §8.1 (incl. missing content-type exception and single overall timeout), §13.3, §13.8, §12.1–12.2; issue 05 (classifiers).
