---
name: tealium-visitor-context
description: >-
  Retrieve live customer context for personalization or an AI agent — either the fast, narrow
  engine-scoped Moments slice (also available as MCP tools) or the full AudienceStream visitor
  profile — and pick the right one for the job.
generated: '2026-08-13'
method: generated
source: >-
  openapi/tealium-personalization-api-openapi.yml, openapi/tealium-customer-api-openapi.yml,
  https://docs.tealium.com/server-side/moments-api/about/,
  https://docs.tealium.com/server-side/moments-api/managed-mcp-server/
api: Tealium Personalization API, Tealium Customer API
operations:
  - getMomentsByVisitorId
  - getMomentsByAttributeValue
  - getVisitorProfile
  - getHistoricalVisitorProfile
---

# Retrieve live visitor context

## Choose the right surface first

| Surface | Operation | Returns | Latency | Auth |
|---|---|---|---|---|
| Moments API | `getMomentsByVisitorId`, `getMomentsByAttributeValue` | Engine-scoped slice: audiences, badges, metrics, properties, flags, dates | ~60ms for payloads ≤1 kB | none — domain allowlist + `Origin`/`Referer` |
| Managed MCP server | `getPersonalizationContentByAnonymousId`, `getPersonalizationContentByVisitorId` | Same as Moments | same | `X-Tealium-Api-Key` |
| Visitor Profile API | `getVisitorProfile`, `getHistoricalVisitorProfile` | Full profile including `attributes` and `currentVisit` | not published | JWT bearer |

**Rule of thumb:** for anything in a request path — personalization, an agent answering a
question — use Moments. Use the Visitor Profile API for back-office work where you need the whole
record. The three responses are *different shapes*, not the same object at different sizes; do not
write code that treats them interchangeably.

## Moments API — by anonymous visitor ID

```
GET https://personalization-api.{region}.prod.tealiumapis.com
    /personalization/accounts/{account}/profiles/{profile}/engines/{engineId}/visitors/{visitorId}
Origin: https://yourdomain.example
Referer: https://yourdomain.example/page
```

Regions: `us-east-1`, `us-west-2`, `eu-central-1`, `ap-southeast-2`. The region must match the
region of the engine.

## Moments API — by known-visitor attribute

```
GET .../engines/{engineId}?attributeId={numericUid}&attributeValue={value}
```

Use this when you know the customer by an identifier you own (email hash, customer ID) rather than
by their anonymous browser ID. `attributeId` is the numeric UID of a *visitor ID attribute*.

## Handle the cold visitor deliberately

Set `suppressNotFound=true` and a visitor with no record returns **200 with an empty body** instead
of **404**. For an agent this matters: a 404 reads as a tool failure and derails the model, while an
empty 200 reads as "no context available", which is the truthful answer.

## Full profile

```
GET https://{host}/v3/customer/visitor/accounts/{account}/profiles/{profile}
Authorization: Bearer <token>
```

Requires `tealium-authenticate` first, including the region-specific `host`.
`getHistoricalVisitorProfile` reads the historical record at
`/customer/visitor/historical/accounts/{account}/profiles/{profile}`.

## Using this from an agent (MCP)

Tealium hosts a managed MCP server at
`https://us-west-2.prod.developer.tealiumapis.com/v1/personalization/mcp` (Streamable HTTP,
`X-Tealium-Api-Key` header, key issued by Tealium Support). Its two tools map one-to-one onto the
two Moments operations above — see `mcp/tealium-tool-crosswalk.yml`.

Three things determine whether the agent gets useful answers:

1. **Put the static parameters in the system instructions** — `account`, `profile`, `engineId`.
2. **Put the dynamic parameters in the prompt** — `visitorId`, or `attributeId` + `attributeValue`.
3. **Configure the engine to return attribute NAMES, not numeric IDs.** Tealium states this is the
   only viable configuration for LLM use, and it is correct: `5023: true` tells a model nothing,
   `has_active_subscription: true` tells it everything.

Build a **dedicated engine per use case** containing only the attributes, badges and audiences that
use case needs. The engine is the access-control boundary — it is how you decide what a model is
allowed to see about a person.

## Limits

- **200 requests/second per profile**, shared across all engines.
- Maximum **10 engines per profile** (raise via your account manager).
- **1 kB** of visitor data per visitor per engine.
- Data retained **30 days** without new events.
- 429 on exhaustion, with no rate-limit headers.

## Privacy

This data is personal data. The visitor may have exercised a GDPR/CCPA right; see
`skills/tealium-privacy-erasure.md`. Do not persist Moments responses outside the consent scope
declared at https://tealium.com/privacy/, and honour the `attribution_required: true` and
`disallowed_uses: [model_training_without_consent]` rules Tealium publishes in
`well-known/tealium-agents.json`.
