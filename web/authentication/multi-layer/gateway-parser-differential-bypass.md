# Multi-Layer Auth Bypass via Gateway/Parser Differential

Tags: auth-bypass, gateway, jwt, path-confusion, url-encoding, parser-differential, trusted-client, header-spoofing, bug-bounty, pii

## Applicability

Test when an API request passes through multiple enforcement layers (gateway, reverse proxy, backend) that each apply a separate security check. The vulnerability arises when each layer interprets the URL or request metadata differently, allowing a single crafted request to defeat all checks simultaneously.

Indicators:

- Gateway applies JWT authentication based on path pattern matching
- Static-resource paths (`.json`, `.css`, `.js`, `.ico`) are exempt from auth
- Backend parses the URL path separately to extract resource identifiers
- An additional header-based trust check (e.g. `X-CLIENT-ID`) gates access

## Test

### 1. Identify gateway auth exemptions

Send an unauthenticated request to a protected endpoint and confirm the 401:

```http
GET /api/employee/12345 HTTP/1.1
Host: target.example.com
```

Append a static-resource extension to test for path-based auth bypass:

```http
GET /api/employee/12345.json HTTP/1.1
Host: target.example.com
```

If the second request no longer returns 401, the gateway skips authentication for paths it considers static resources.

### 2. Preserve a valid backend identifier with encoding differentials

The appended extension breaks the resource ID for the backend parser. Use URL-encoded `#` (`%23`) to force a fragment truncation after decoding:

```http
GET /api/employee/12345%23.json HTTP/1.1
Host: target.example.com
```

Processing at each layer:

| Layer   | Sees              | Interprets as |
|---------|--------------------|---------------|
| Gateway | `12345%23.json`    | `.json` path — skip auth |
| Backend | `12345#.json`      | Truncates at `#` → `12345` |

The gateway matches on the raw path, the backend decodes and truncates, and the resource ID arrives clean.

### 3. Satisfy trusted-client header checks

If the endpoint also enforces a client-identity header, add it:

```http
GET /api/employee/12345%23.json HTTP/1.1
Host: target.example.com
X-CLIENT-ID: WEBAPP
```

Discover valid header values by inspecting legitimate browser traffic, JavaScript source, or mobile app bundles.

### 4. Full exploit request

```http
GET /api/employee/12345%23.json HTTP/1.1
Host: target.example.com
X-CLIENT-ID: WEBAPP
```

Chain summary:

1. `.json` suffix → gateway treats path as static, skips JWT auth
2. `%23` → decoded to `#` by backend, truncating the extension and yielding a clean ID
3. `X-CLIENT-ID: WEBAPP` → satisfies the trusted-client gate

## Verify

Confirm the bypass by comparing responses:

- Unauthenticated request to `/api/employee/12345` returns `401`
- Crafted request to `/api/employee/12345%23.json` with the client header returns `200` with employee data
- Iterate over sequential or enumerable IDs to confirm access to records beyond the attacker's own, establishing mass PII exposure

## Variants

- **Other static extensions**: `.css`, `.js`, `.ico`, `.xml`, `.woff`, `.map` — test each against the gateway's bypass list
- **Alternative fragment/truncation characters**: `%23` (`#`), `%3F` (`?`), `%00` (null byte), `%0A` (newline) — different parsers truncate on different delimiters
- **Double encoding**: `%2523` if the gateway decodes once and the backend decodes again
- **Path traversal combinations**: `/api/employee/12345%23.json/../../other-resource` when the backend normalizes after truncation
- **Mixed case headers**: `x-client-id`, `X-Client-Id` — test whether header matching is case-sensitive

## Report

Three independent security checks defeated by a single request exploiting interpretation differences between the gateway and backend:

1. **Gateway JWT bypass** — path-pattern matching treats `.json` suffix as a static resource, skipping authentication entirely
2. **Backend parser differential** — URL-encoded `#` (`%23`) causes post-decode truncation, removing the appended extension and restoring a valid resource ID
3. **Trusted-client header spoofing** — hardcoded client identifier discovered in client-side code and replayed

Impact: unauthenticated access to any employee record by ID enumeration, resulting in mass PII exfiltration. Demonstrate with two or more distinct employee records accessed without any valid session.
