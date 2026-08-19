---
name: tealium-authenticate
description: >-
  Exchange a Tealium username and API key for a JWT bearer token, capture the region-specific host
  returned alongside it, and reuse both correctly for the rest of the session. Every other Tealium
  V3 skill depends on this one.
generated: '2026-08-13'
method: generated
source: openapi/tealium-auth-api-openapi.yml, https://docs.tealium.com/api/v3/getting-started/authentication/
api: Tealium Auth API
operations:
  - generateBearerToken
---

# Authenticate against the Tealium V3 platform

## When to use this

Before any call to the Collect, Visitor Profile, Visitor Privacy or iQ Profiles APIs. The Moments
API and the managed MCP server do NOT use this token — they use domain allowlisting and a separate
`X-Tealium-Api-Key` respectively.

## Prerequisites

- A Tealium account name and profile name (the default profile is conventionally `main`).
- A username (an email address) and an API key generated in Tealium iQ.
  See https://docs.tealium.com/administration/security-access/api-keys/.

## Steps

1. **Exchange the key for a token** — `generateBearerToken`

   ```
   POST https://platform.tealiumapis.com/v3/auth/accounts/{account}/profiles/{profile}
   Content-Type: application/x-www-form-urlencoded

   username={email}&key={api_key}
   ```

   Both values must be URL-encoded. API keys contain punctuation that will break the request if
   sent raw.

2. **Read BOTH fields of the response.** The body is:

   ```json
   { "token": "<jwt>", "host": "us-east-1-platform.tealiumapis.com" }
   ```

   `token` is the bearer credential. `host` is the region your account's data lives in, and it is
   just as load-bearing. This is the single most-missed step on the platform.

3. **Send every subsequent region-specific call to the returned host**, not to
   `platform.tealiumapis.com`:

   ```
   GET https://{host}/v3/customer/visitor/accounts/{account}/profiles/{profile}
   Authorization: Bearer <token>
   ```

   This is why the OpenAPI `servers[]` blocks for the data APIs are templated as
   `https://{host}/v3` — a concrete host would be wrong for most accounts.

4. **Cache the token and reuse it.** It is valid for **30 minutes** (1800 seconds).

## Rules

- **Never mint a token per request.** Tealium's documentation is explicit: "Failure to use the
  token in this manner will result in authentication call throttling at Tealium's discretion."
  Token churn is a throttling trigger, not merely wasteful.
- Refresh at ~25 minutes, or lazily on the first `401`.
- Cache the `host` with the token; they expire together as a pair.
- Never log the token or the API key.

## Long-lived tokens (SCIM only)

For SCIM provisioning, mint a 90-day token from a different host:

```
POST https://developer.tealiumapis.com/v2/auth-long-lived/token
Content-Type: application/x-www-form-urlencoded

account={ACCOUNT}&profile=main&username={USER}&key={API_KEY}
```

Returns `access_token`, `token_type: Bearer`, `expires_in: 7776000`, `scope: "profile email"`.
Rate limit: **10 requests per minute per IP**. Revoke with
`POST https://developer.tealiumapis.com/v2/auth-long-lived/revoke` (20 req/min per IP) — revocation
is immediate and permanent, and returns `200` whether or not the token was already invalid.

Use a dedicated service user (for example `scim@example.com`) rather than a person's account, so
the integration survives that person leaving.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | Missing username or key | Check both form fields are present and URL-encoded |
| 401 | Invalid credentials, or expired token | Re-exchange once; a second 401 is a credential problem |
| 429 | Auth throttling | You are minting tokens too often — cache and reuse |

No `Retry-After` or `RateLimit-*` header is returned on 429. Back off exponentially and blindly.
See `errors/tealium-problem-types.yml` and `conventions/tealium-conventions.yml`.
