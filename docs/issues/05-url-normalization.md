# URL validation & normalization library

## Summary

Implement `src/capture/url.ts`: strict URL validation (capture boundary), deterministic normalization for the dedupe key, and private/loopback address classification helpers shared with the fetcher's egress policy. Pure functions, no I/O.

## Context

Every article enters through `earmark add` or the iCloud inbox; both must apply identical validation (DESIGN §7.1) and the same dedupe key (§7.2). The fetcher (issue 10) enforces the private-network egress policy (§13.3) and needs IP-range classification helpers that belong here to keep one source of truth.

## Scope

- `src/capture/url.ts` only, plus tests. No DNS, no network.

## Detailed Requirements

1. `validateCaptureUrl(raw: string): {ok: URL} | {error: string}` — rules of DESIGN §7.1:
   - must parse with `new URL(raw)`; scheme exactly `http:` or `https:`; hostname non-empty
   - reject embedded credentials (`url.username || url.password`)
   - reject `raw.length > 2048`
   - reject whitespace-containing raw strings (trim first; inner whitespace → invalid)
   - error strings are stable snake_case reasons (`invalid_format`, `unsupported_scheme`, `has_credentials`, `too_long`) — callers wrap into `EARMARK_URL_INVALID`.
2. `normalizeUrl(u: URL | string): string` — exactly DESIGN §7.2 steps 1–6:
   - lowercase protocol+hostname; strip `:80` (http) / `:443` (https)
   - drop fragment
   - remove tracking params by exact name: `utm_source, utm_medium, utm_campaign, utm_term, utm_content, gclid, fbclid, igshid, mc_cid, mc_eid, ref_src, cmpid` (all occurrences)
   - sort remaining params by name; duplicate names keep original relative order (stable sort)
   - remove trailing `/` when path length > 1
   - serialize via `URL#toString` conventions (IDN stays punycode; percent-encoding as `URL` produces; empty query → no `?`)
   - Deterministic: same input string always yields the same output (property test with shuffled param insertion order).
3. Address classification (used by issue 10; no DNS here):
   - `isPrivateIPv4(ip)`: 10/8, 172.16/12, 192.168/16, 127/8, 169.254/16, 0.0.0.0
   - `isPrivateIPv6(ip)`: `::1`, `fc00::/7`, `fe80::/10`, IPv4-mapped forms re-checked as v4 (`::ffff:a.b.c.d`)
   - `isPrivateHostname(name)`: `localhost`, `*.localhost`, `*.local` (case-insensitive)
   - `isBlockedAddress(ipOrName): boolean` combining the above; malformed IP strings → `true` (fail closed).
4. No external deps: implement IPv4/IPv6 parsing with `net.isIP` + manual range math (document the fc00::/7 and fe80::/10 mask logic in code).
5. Export a frozen `TRACKING_PARAMS` array (issue 29 and docs reference it).

## Acceptance Criteria

- [ ] Validation table passes: `javascript:alert(1)`, `file:///etc/passwd`, `data:text/html,x`, `ftp://x`, `http://user:pw@host/`, 2049-char URL, `http://` (empty host), `not a url` → each rejected with the specified reason; plain http/https accepted.
- [ ] Normalization examples all hold:
   - `HTTPS://Example.COM:443/Post/?b=2&utm_source=x&a=1#frag` → `https://example.com/Post?a=1&b=2`
   - `http://example.com/` → `http://example.com/` (root path kept)
   - `https://example.com/a/` → `https://example.com/a`
   - `https://日本語.example/x` → punycoded host form
- [ ] Property test: for 100 randomized param orders of the same param set, normalized output identical.
- [ ] IP classification table passes incl. `172.15.255.255` (public), `172.16.0.0` (private), `::ffff:192.168.0.1` (private), `fe80::1` (private), `2001:db8::1` (public-range for tests), `garbage` (blocked).

## Validation

`vitest` table-driven + property tests as above; 100% line coverage on this file (it is small and security-relevant). No manual steps.

## Dependencies

01.

## Non-goals

DNS resolution and redirect-hop policy (issue 10); dedupe DB lookup (issue 06); URL un-shortening or canonical-link discovery (v2 candidates).

## Design References

DESIGN §7.1, §7.2, §13.3; issue 29 (verification suite reuses these tables).
