---
name: tealium-privacy-erasure
description: >-
  Answer a GDPR/CCPA data subject access request and execute an erasure against the Tealium
  Customer Data Hub — look up everything held about a visitor, submit the deletion, and poll the
  transaction to completion.
generated: '2026-08-13'
method: generated
source: openapi/tealium-privacy-api-openapi.yml, https://docs.tealium.com/api/v3/visitor-privacy/about/
api: Tealium Privacy API
operations:
  - getVisitorIdAttributes
  - getPrivacyVisitor
  - deletePrivacyVisitor
  - getDeleteTransactionStatus
---

# Handle a data subject access or erasure request

## When to use this

A person has asked what you hold about them (DSAR / right of access) or asked you to delete it
(right to erasure / right to delete). This API exists specifically to satisfy those obligations
against the Customer Data Hub.

## Prerequisites

Run `tealium-authenticate` first; use the region-specific `host` it returned.

## Steps

1. **Find the right lookup key** — `getVisitorIdAttributes`

   ```
   GET https://{host}/v3/privacy/visitor/accounts/{account}/profiles/{profile}/ids
   Authorization: Bearer <token>
   ```

   Returns the visitor ID attributes configured on the profile, so you know which `attributeId` to
   search by — email, customer ID, loyalty number, whatever this account uses.

2. **Retrieve everything held about the visitor** — `getPrivacyVisitor`

   ```
   GET https://{host}/v3/privacy/visitor/accounts/{account}/profiles/{profile}
       ?attributeId={numericUid}&attributeValue={value}
   Authorization: Bearer <token>
   ```

   Returns `audiences`, `badges` and `attributes`. This is the payload you disclose in a DSAR
   response. A **404** means no record exists for that identifier — which is itself a valid, and
   disclosable, answer.

3. **Submit the deletion** — `deletePrivacyVisitor`

   ```
   DELETE https://{host}/v3/privacy/visitor/accounts/{account}/profiles/{profile}
          ?attributeId={numericUid}&attributeValue={value}
   Authorization: Bearer <token>
   ```

   Returns a `transactionId`. **The deletion is asynchronous.** A 2xx here means accepted, not done.

4. **Poll to completion** — `getDeleteTransactionStatus`

   ```
   GET https://{host}/v3/privacy/visitor/accounts/{account}/profiles/{profile}
       /transactions/{transaction_id}
   Authorization: Bearer <token>
   ```

   Deletions complete **within 30 days**. Poll on a schedule measured in hours or days, not
   seconds, and record the transaction ID in your compliance log as the evidence of the request.

## Rules

- **50 requests per second** on this API, and Tealium states plainly it is "not designed for
  high-volume usage". It is a compliance instrument, not a bulk data pipe. Do not use it to export
  profiles.
- The 30-day completion window is Tealium's, and it sits *inside* your own statutory clock
  (one month under GDPR Art. 12(3), 45 days under CCPA). Submit the deletion early in your window,
  not on the last day.
- **Deleting from Tealium is not deleting from your stack.** Tealium forwards data to connectors
  and destinations; erasure here does not propagate to the downstream systems those connectors fed.
  Enumerate them separately.
- Log the identifier searched, the transaction ID and the terminal status. Do not log the returned
  profile payload.
- There is no idempotency key. Re-submitting a deletion for the same visitor issues a *new*
  transaction; that is harmless but produces duplicate audit rows — key your own log on
  (attributeId, attributeValue) and store the first transaction ID.

## Errors

| Status | Cause | Action |
|---|---|---|
| 400 | Missing `attributeId` or `attributeValue` | Both are required on lookup and delete |
| 401 | Expired or invalid bearer token | Refresh once |
| 404 | Visitor not found, or transaction ID not found | For a lookup this is a legitimate "we hold nothing" |
| 429 | Over 50 requests/second | Back off; no `Retry-After` is returned |

See `errors/tealium-problem-types.yml`, `conformance/tealium-conformance.yml` (GDPR/CCPA) and
`lifecycle/tealium-lifecycle.yml` (retention windows).
