# Web cache deception through cache/origin route confusion

**Tags:** web-cache-deception, WCD, CDN, cache-key, path-normalization, delimiter, static-suffix, CloudFront, Akamai, Fastly, Cloudflare, authenticated-content

## Purpose

Test whether an attacker can make a CDN store a response generated for an authenticated page and then retrieve that response without the victim's credentials. This is different from cache poisoning: changing an unkeyed input or caching a redirect does not prove cache deception unless the cached object contains private, victim-specific data.

## Applicability

Prioritize this technique when all three conditions are plausible:

1. The site has an authenticated route that returns data worth reporting, such as account details, tokens, private files, or administrative state.
2. The edge has a cache rule for static-looking paths, directories, or extensions.
3. The CDN and origin disagree about the effective route or path after delimiter parsing, rewriting, decoding, or normalization.

Use only an owned test session to seed the cache. A random path marker creates an isolated cache object and avoids retrieving another user's content.

Inspect authenticated 404 and other fallback templates as well as status-200 account pages. Real-world cases have leaked usernames, API keys, CSRF tokens, and profile data from personalized error templates, and a leaked action-authorizing token can raise the impact to account takeover.

## Required success condition

A reportable result requires this complete chain:

1. With the owned session, the crafted static-looking URL returns the same private representation as the ordinary authenticated route.
2. Without any session cookie or authorization header, the exact crafted URL returns that seeded private representation.
3. A cache signal or a same-key behavioral control proves the unauthenticated response came from a shared cache rather than the origin.
4. A fresh, unseeded marker does not return that private representation to an unauthenticated request.

Do not promote a result based only on `Cache-Control`, a reflected marker, an open redirect, or a cache `MISS` header.

## Map the two path parsers first

Build separate models for the origin route and the cache rule.

- **Origin delimiter test:** append a unique segment after candidate delimiters. If the authenticated response remains equivalent to the ordinary private route, the origin discarded, terminated, or rewrote the suffix.
- **Cache delimiter test:** request harmless public URLs with and without the same suffix, then compare cache hits and cache keys. Identify which characters the cache treats as part of a static suffix.
- **Normalization test:** place dot segments, encoded separators, or an application rewrite before the static suffix. Test whether the CDN selects a cache behavior using one path while the origin routes another.

Start with `/`, `;`, `.`, `?`, `%3f`, `%23`, and `%3b`, subject to the application and CDN accepting those characters. A fragment introduced as a literal `#` is not sent by a browser; test its encoded form if relevant. Do not assume a delimiter from a framework fingerprint—prove each layer's behavior.

## Candidate construction

Use a long random marker and one static extension at a time:

```text
/account;<marker>.css
/account/<marker>.css
/account<delimiter><marker>.js
/<static-directory>/../account/<marker>.css
```

Common cache-rule probes are `.css`, `.js`, `.ico`, `.woff2`, `.jpg`, `.png`, and a known cached directory. Match extensions actually used by the target instead of sending a large generic wordlist.

When a CDN normalizes paths before selecting a cache behavior, also compare a path that normalizes into a private route with the raw path that the origin receives. CloudFront explicitly normalizes the URI to match cache behaviors but forwards the raw URI path to the origin, so overlapping path patterns deserve special attention.

## Low-request verification sequence

Use the same method, body, and non-auth headers in all requests. Strip volatile values such as `Date`, request IDs, and dynamic timestamps before comparing response hashes.

```text
A. GET /account with owned session
   Expect: private representation P, no forced static suffix

B. GET /account;<marker-A>.css with owned session
   Expect: representation equivalent to P and a cache MISS/store signal

C. GET /account;<marker-A>.css with no session
   Expect for a finding: P and a HIT/positive Age/same cached object

D. GET /account;<marker-B>.css with no session
   Expect as control: login, denial, or public representation, not P
```

If the response exposes a fresh anti-CSRF value or nonce tied to the owned session, use it as the benign personalization marker. Otherwise compare stable account-specific fields from the owned account. Never use a marker copied from another user.

For an edge that emits no cache-status headers, prime an isolated marker twice, wait at least one second, then compare stable `Date`, `Age`, ETag, body, and timing with an unseeded control. Require the authenticated-to-unauthenticated representation leak; timing alone is insufficient.

## Akamai checks

- `TCP_HIT`, `TCP_MEM_HIT`, or positive `Age` supports shared cache delivery; repeated `TCP_MISS` does not.
- Treat `X-True-Cache-Key` only as a cache-key observation. A shared object still must leak across the owned authenticated and unauthenticated contexts.
- Check whether the complete query string and random path marker are in the key. Analytics-parameter reflection with a different cache key is not poisoning or deception.

## Cloudflare checks

- Origin Cache-Control normally prevents shared storage of authenticated/private content, but Cache Rules and Workers can override behavior.
- Cache Deception Armor validates that the URL extension agrees with the response `Content-Type`. If that control is enabled, test non-extension rules, static directories, Workers/rewrite disagreement, and normalization rather than spraying more extensions.
- `Set-Cookie`, `private`, and `no-store` typically prevent default CDN caching; a real cross-context replay is necessary to prove that a rule overrode them.

## CloudFront checks

- Record the path pattern of the selected cache behavior, then test which raw path the origin receives.
- Inspect the cache policy, origin request policy, and cache key separately when configuration is available. Query strings, cookies, and headers can be excluded from the cache key or not forwarded at all.
- A policy that forwards an authorization-bearing cookie but excludes it from the cache key creates risk only if the response is actually cacheable and replayable.

## Fastly checks

- Fastly's normal `Vary: Cookie` behavior does not store a zero-TTL personalized response, while strict `req.hash_always_miss` and shielding settings can lead to private object storage patterns.
- Confirm with an owned session and an unauthenticated replay on one unique key. A `Vary` mismatch without the private response body is not enough.

## Stop and reject conditions

- The authenticated crafted path redirects to login or returns a static error page.
- The unauthenticated response is generated by the origin or only reflects values present in that request.
- Both seeded and unseeded markers expose the same public content.
- The cache key contains the entire unique suffix and no cross-context replay occurs.
- The only impact is a cached 301/302, generic denial, or public HTML.
- The only signal is cache eligibility metadata without a cache hit.

## Reporting evidence

Include the normal private route, the exact isolated cache URL, the owned-session seed response, the cookie-less replay, cache signal, fresh-marker negative control, redacted private fields, cache duration, and CDN identified. Explain the delimiter or normalization disagreement and why the victim is realistically induced to visit the crafted URL.

## Primary references

- Omer Gil, original Web Cache Deception whitepaper: https://whitepaper.securing.dev/
- PortSwigger Web Security Academy, web cache deception: https://portswigger.net/web-security/web-cache-deception
- PortSwigger research, cache key normalization and web cache poisoning: https://portswigger.net/research/web-cache-poisoning-without-keyed-input
- Cloudflare Cache Deception Armor: https://blog.cloudflare.com/cache-deception-armor/
- AWS CloudFront path-pattern normalization: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html
- Fastly personalized/private content caching: https://www.fastly.com/documentation/guides/full-site-delivery/caching/about-cache-control-headers/
- HackerOne #1271944, personalized 404 pages leaked profile data and CSRF tokens: https://hackerone.com/reports/1271944
- HackerOne #260697, leaked CSRF token chained to an email change: https://hackerone.com/reports/260697
