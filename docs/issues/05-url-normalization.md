# URL validation & normalization library

## Summary

Implement `src/capture/url.ts`: strict URL validation (capture boundary), deterministic normalization for the dedupe key, and private/loopback address classification helpers shared with the fetcher's egress policy. Pure functions, no I/O.

## Context

Every article enters through `earmark add` or the iCloud inbox; both must apply identical validation (DESIGN §7.1) and the same dedupe key (§7.2). The fetcher (issue 10) enforces the private-network egress policy (§13.3) and needs IP-range classification helpers that belong here to keep one source of truth.

## Scope

- `src/capture/url.ts` only, plus tests. No DNS, no network.

## Detailed Requirements

1. `validateCaptureUrl(raw: string): {ok: URL} | {error: string}` — rules of DESIGN §7.1, checked in this exact order (first failure wins):
   1. trim `raw`; empty → `invalid_format`
   2. length after trim > 2048 → `too_long`
   3. contains whitespace (`/\s/`) after trim → `contains_whitespace`
   4. `new URL(raw)` throws → `invalid_format`
   5. scheme not exactly `http:`/`https:` → `unsupported_scheme`
   6. empty hostname → `invalid_format`
   7. `url.username || url.password` → `has_credentials`
   - error strings are stable snake_case reasons; callers wrap into `URL_INVALID`.
2. `normalizeUrl(u: URL | string): string` — exactly DESIGN §7.2 steps 1–6:
   - lowercase protocol+hostname; strip `:80` (http) / `:443` (https)
   - drop fragment
   - remove tracking params by exact name: `utm_source, utm_medium, utm_campaign, utm_term, utm_content, gclid, fbclid, igshid, mc_cid, mc_eid, ref_src, cmpid` (all occurrences)
   - sort remaining params by name; duplicate names keep original relative order (stable sort)
   - remove trailing `/` when path length > 1
   - serialize via `URL#toString` conventions (IDN stays punycode; percent-encoding as `URL` produces; empty query → no `?`)
   - Deterministic: same input string always yields the same output (property test with shuffled param insertion order).
3. Address classification (used by issue 10; no DNS here) — two separate functions with distinct contracts (a hostname is never treated as a malformed IP):
   - `isBlockedIp(ip: string): boolean` — input is expected to be an IP literal (typically a resolved address). Returns `true` ("not public Internet") for the full special-use set, `false` only for public unicast. Malformed input (`net.isIP(ip) === 0`) → `true` (fail closed — resolved addresses are always well-formed, so a malformed value signals a bug upstream).
     - IPv4 blocked ranges: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10` (CGNAT), `127.0.0.0/8`, `169.254.0.0/16`, `172.16.0.0/12`, `192.0.0.0/24`, `192.0.2.0/24`, `192.168.0.0/16`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, `224.0.0.0/4` (multicast), `240.0.0.0/4` (reserved incl. broadcast)
     - IPv6 blocked ranges: `::/128` (unspecified), `::1/128`, `::ffff:0:0/96` (IPv4-mapped — extract the v4 and re-check with the v4 table), `64:ff9b::/96` (NAT64 — re-check embedded v4), `100::/64` (discard), `2001:db8::/32` (doc), `fc00::/7`, `fe80::/10`, `ff00::/8` (multicast)
   - `isPrivateHostname(name: string): boolean` — `localhost`, `*.localhost`, `*.local` (case-insensitive). Any other name → `false` (public DNS names are vetted by the fetcher via resolution + `isBlockedIp`, DESIGN §13.3).
4. No external deps: implement IPv4/IPv6 parsing with `net.isIP` + manual range math (document the mask logic in code; IPv6 compared on the 128-bit value from expanded hextets).
5. Export frozen constants: `TRACKING_PARAMS` (issue 29 and docs reference it) and the blocked-range tables (doctor/docs may display them).

## Acceptance Criteria

- [ ] Validation table passes: `javascript:alert(1)`, `file:///etc/passwd`, `data:text/html,x`, `ftp://x`, `http://user:pw@host/`, 2049-char URL, `http://` (empty host), `not a url`, `http://a b.com/x` (inner whitespace → `contains_whitespace`), `"  https://ok.example/  "` (leading/trailing whitespace → accepted after trim) → each with the specified reason; plain http/https accepted.
- [ ] Normalization examples all hold:
   - `HTTPS://Example.COM:443/Post/?b=2&utm_source=x&a=1#frag` → `https://example.com/Post?a=1&b=2`
   - `http://example.com/` → `http://example.com/` (root path kept)
   - `https://example.com/a/` → `https://example.com/a`
   - `https://日本語.example/x` → punycoded host form
- [ ] Property test: for 100 randomized orderings of a param set with **unique names**, normalized output identical; separate example asserts duplicate-name params keep original relative order after sorting by name.
- [ ] `isBlockedIp` table passes — blocked: `172.16.0.0`, `100.64.0.1`, `0.0.0.0`, `198.18.0.1`, `224.0.0.1`, `255.255.255.255`, `::1`, `::`, `::ffff:192.168.0.1`, `64:ff9b::c000:201` (embedded `192.0.2.1`), `fe80::1`, `ff02::1`, `2001:db8::1`, `garbage`; public: `172.15.255.255`, `100.63.255.255`, `8.8.8.8`, `93.184.216.34`, `2600::1`, `::ffff:8.8.8.8`.
- [ ] `isPrivateHostname`: `localhost`, `LOCALHOST`, `foo.localhost`, `printer.local` → true; `example.com`, `local.example.com` → false.

## Validation

`vitest` table-driven + property tests as above; 100% line coverage on this file (it is small and security-relevant). No manual steps.

## Dependencies

01.

## Non-goals

DNS resolution and redirect-hop policy (issue 10); dedupe DB lookup (issue 06); URL un-shortening or canonical-link discovery (v2 candidates).

## Design References

DESIGN §7.1, §7.2, §13.3; issue 29 (verification suite reuses these tables).
