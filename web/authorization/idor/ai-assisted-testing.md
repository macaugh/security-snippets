# AI-Assisted IDOR Testing with Captured HTTP Traffic

Tags: idor, bola, broken-access-control, authorization, burp-mcp, proxy, api

## Inputs

Prepare at least two authorized test accounts, their independent sessions, and known object identifiers owned by each account. Record role differences and the flow that creates each object.

## Discovery

Search captured HTTP traffic for identifiers in paths, queries, bodies, headers, and filenames. Prioritize requests that retrieve, update, or delete user resources. Include numeric IDs, UUIDs, usernames, account IDs, order IDs, transaction IDs, and indirect references.

## Test

Send each request with the attacker account's session while substituting only an identifier owned by the second account. Test read, write, and delete operations independently. Preserve unrelated headers and workflow state. Compare with an attacker-owned baseline and an invalid-object control.

## Validate

Confirm findings with manual replay. Save the original request, mutated request, and minimum response evidence proving cross-account data access or state change. A status difference alone is insufficient.

In RedAmon, use captured HTTP history and `proxy_brain` to search, replay, and compare requests. In Burp environments, create Repeater tabs through Burp MCP for manual confirmation.
