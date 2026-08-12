---
name: Subscribe to Decisiv SRM webhook events
description: >-
  Register a webhook endpoint against an SRM account, choose which service events to receive, rotate the
  signing key, and suppress your own echo during bulk writes.
api: openapi/decisiv-account-management-openapi.yml
base_url: https://srm-api.decisivapps.com
operations:
  - getWebhooksEventsbyAccountId
  - createWebhookByAccountId
  - getWebhooks
  - getWebhooksById
  - updateWebhookbyId
  - deleteWebhooksbyId
  - refreshSigningKeyWebhooksbyId
generated: '2026-08-12'
method: generated
---

# Subscribe to SRM service events (Decisiv SRM Gateway)

Polling `listCases` is the wrong way to follow a service event. Decisiv manages webhooks through the
Account Management module. Grounded in `openapi/decisiv-account-management-openapi.yml`.

## Steps

1. **Ask what you may subscribe to.** `getWebhooksEventsbyAccountId`
   (`GET /account_management/v1/accounts/{account_id}/webhook_events`). This is the authoritative list
   **for that account** — event availability is account-scoped, so do not hard-code the catalog.
   Subscribing to an event that is not available returns **422**
   `decisiv:webhook_events:001`.

2. **Register the endpoint.** `createWebhookByAccountId`
   (`POST /account_management/v1/accounts/{account_id}/webhooks`) with:
   - `url` — **must** be `https://` (schema pattern `^https://`). A URL already registered on the
     account is **422** `decisiv:webhooks:001`; one webhook per URL.
   - `events` — at least one canonical event name.
   - `custom_headers` — optional, **maximum 10** (`decisiv:webhooks:003`). You may not set
     `authorization`, `content-type`, `host` or `user-agent`
     (`decisiv:webhooks:002`), so do not try to authenticate deliveries with a bearer header — use the
     signing key.
   - `enabled` — defaults to `true`.

3. **Verify deliveries with the signing key.** Each webhook carries a signing key. Rotate it with
   `refreshSigningKeyWebhooksbyId`
   (`POST /account_management/v1/accounts/{account_id}/webhooks/{id}/refresh_signing_key`), which
   returns the webhook with a new key. Decisiv does not publish the signature header name or algorithm
   in the public spec — get it from your Decisiv contact at provisioning time and pin it.

4. **Maintain.** `getWebhooks` / `getWebhooksById` to inspect, `updateWebhookbyId` (PATCH) to change the
   URL, event set, headers or `enabled` flag, `deleteWebhooksbyId` to remove.

## The event catalog

Names Decisiv publishes in its own examples (18):

| Domain | Events |
|---|---|
| `asset_management` / asset | `asset:registered`, `asset:transferred`, `asset:meter_updated` |
| `asset_management` / case | `case:created`, `case:closed`, `case:reopened`, `case:note_posted`, `case:attachment_posted`, `case:repair_status_changed` |
| `asset_management` / estimate | `estimate:approval_requested`, `estimate:approved`, `estimate:declined` |
| `asset_management` / service request | `service_request:accepted`, `service_request:declined` |
| `maintenance` | `service_event:created`, `scheduled_operation:planned`, `scheduled_operation:overdue`, `scheduled_operation:invalidated` |

All are prefixed `decisiv:` — e.g. `decisiv:asset_management:estimate:approval_requested`.

## Silence your own writes

When your integration writes in bulk, send `X-DECISIV-SILENCE-EVENTS` on the write request listing the
events to mute for that transaction. Without it a backfill fans thousands of deliveries back at your own
endpoint. Invalid values return **400** `decisiv:silence_webhook_events:001`.

## Delivery semantics you must design for

Decisiv publishes no AsyncAPI document, and no delivery guarantees for the Gateway webhooks. What it
*does* publish — for the legacy Platform API push notifications, at
`https://api-docs.decisiv.net/docs/api/1/architecture/notifications/` — is an unusually candid account of
the failure mode, and it is the right mental model for any Decisiv event feed:

- Notifications **may arrive out of order**, both from concurrent consumer processes and from retried
  timeouts. A `note_posted` can land before the `case:created` it belongs to.
- The legacy feed defaults to a **5 second timeout** and **5 retries**, after which the message is
  **emailed** to the profile user and dropped from the queue.
- Decisiv's own guidance is to lock on the parent resource and fetch it from the API when a child event
  arrives for something you have not seen.

Design accordingly: treat every event as a **hint to re-read**, not as state. On any event, `getCase`
(or the relevant GET) and reconcile against what you hold. Make your handler idempotent — you will get
duplicates.
