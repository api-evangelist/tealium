---
name: tealium-collect-events
description: >-
  Send single, bulk, regional and data-source-scoped events into the Tealium Customer Data Hub via
  the Collect HTTP API, within the 100 events/second budget and without a safety net for retries.
generated: '2026-08-13'
method: generated
source: openapi/tealium-collect-api-openapi.yml, https://docs.tealium.com/api/v3/http-api/about/
api: Tealium Collect API
operations:
  - collectEventPost
  - collectEventGet
  - collectBulkEvent
  - collectRegionalEvent
  - collectRegionalBulkEvent
  - collectIntegrationEvent
---

# Send events into the Tealium Customer Data Hub

## When to use this

To get behavioural or transactional events from any server, app or system into EventStream /
AudienceStream, where they update the visitor profile that the Visitor Profile and Moments APIs
later read.

## Prerequisites

Run `tealium-authenticate` first. You need the JWT and the region-specific `host` it returned.

## Choose the operation

| Operation | Use when |
|---|---|
| `collectEventPost` | One event, JSON body. The default. |
| `collectEventGet` | One event delivered as query parameters. Constrained payloads only. |
| `collectBulkEvent` | Up to **10** events in one request with a shared field block. |
| `collectRegionalEvent` | One event pinned to a specific region. |
| `collectRegionalBulkEvent` | Bulk, pinned to a specific region. |
| `collectIntegrationEvent` | Event scoped to a named data source: `/collect/integration/event/accounts/{account}/profiles/{profile}/datasources/{datasourceId}` |

## Steps

1. **Build the payload.** The required identity fields are:

   - `tealium_account` — your account name
   - `tealium_profile` — your profile name
   - `tealium_event` — the event name (this is what triggers rules downstream)
   - `tealium_visitor_id` — the visitor this event belongs to
   - `tealium_datasource` — the data source ID
   - `tealium_environment` — `dev`, `qa` or `prod`

   Everything else in the object becomes data-layer attributes.

2. **Send it.**

   ```
   POST https://{host}/v3/collect/event
   Authorization: Bearer <token>
   Content-Type: application/json
   ```

3. **For bulk**, wrap the events with the fields they have in common:

   ```json
   { "shared": { "tealium_account": "...", "tealium_profile": "..." },
     "events": [ { "tealium_event": "purchase" }, { "tealium_event": "page_view" } ] }
   ```

   Maximum **10** events per request. See `examples/tealium-collect-bulk-event-example.json`.

4. **Verify with Trace, not by reading back.** There is no GET for an event and no event ID is
   returned. Start a trace, attach the trace ID, send the event, and inspect the trace log.
   See `sandbox/tealium-sandbox.yml`.

## Rate limit and retry — read this before writing a retry loop

- **100 events per second per account.** A bulk call carrying 10 events counts as **10**, not 1.
- Exhaustion returns **429** with **no** `Retry-After` and **no** `RateLimit-*` headers. You cannot
  read your remaining budget; you can only observe the failure.
- **There is no idempotency key.** Tealium publishes no `Idempotency-Key` header and no
  deduplication field. A retried `POST /collect/event` creates a **second event** in the Customer
  Data Hub, which will re-fire audience and connector rules.

  Therefore: retry only on `429` and `5xx` with exponential backoff, treat any `2xx` as final, and
  never retry after an ambiguous timeout without a dedupe key of your own carried as a data-layer
  attribute that downstream rules can filter on.

- The Data Connect API has its own budget: 500 events/second, 500 records/second, 2.5 MB maximum
  request, auto-batched at 500 records.

## Errors

| Status | Cause | Action |
|---|---|---|
| 400 | Missing required fields, invalid JSON keys, empty body | Fix the payload; do not retry unchanged |
| 401 | Expired or invalid bearer token | Refresh once via `tealium-authenticate` |
| 429 | Over 100 events/second | Exponential backoff; consider bulk to reduce request count (but note events still count individually) |

See `errors/tealium-problem-types.yml` and `rate-limits/tealium-rate-limits.yml`.
