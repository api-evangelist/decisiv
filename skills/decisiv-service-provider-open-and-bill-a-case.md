---
name: Open a service case for a customer asset and build the repair order
description: >-
  As a service provider or dealer, find the customer's asset, open a case idempotently, build the line
  items, parts, additional charges and technician time, then close it — using the Decisiv SRM Gateway
  Service Management API.
api: openapi/decisiv-service-management-openapi.yml
base_url: https://srm-api.decisivapps.com
operations:
  - listCases
  - getCase
  - updateCase
  - updateCaseMeterData
  - listCaseLineItems
  - createCaseLineItem
  - getCaseLineItem
  - updateCaseLineItem
  - deleteCaseLineItem
paths:
  - POST /service_management/{srm_account_id}/v1/customer_assets/{id}/create_case
  - GET /service_management/{srm_account_id}/v1/customer_assets
  - POST /service_management/{srm_account_id}/v1/customer_assets
  - GET /service_management/{srm_account_id}/v1/customer_assets/{id}/service_history
  - GET /service_management/{srm_account_id}/v1/customers
  - POST /service_management/{srm_account_id}/v1/customers
  - POST /service_management/{srm_account_id}/v1/cases/{case_id}/actions/open
  - POST /service_management/{srm_account_id}/v1/cases/{case_id}/actions/close
  - POST /service_management/{srm_account_id}/v1/time_tasks
generated: '2026-08-12'
method: generated
---

# Open and bill a service case (Decisiv SRM Gateway)

The service-provider side of the same service event. Grounded in
`openapi/decisiv-service-management-openapi.yml` (104 paths, 139 operations).

> **Read this first.** Only a minority of Service Management operations carry an `operationId` — most are
> addressable only by method + path. Where an id exists it is named below; where it does not, the path is
> given verbatim. Do not invent operationIds for this module.

## Before you start

Same auth and provisioning rules as the fleet skill: OAuth 2.0 authorization code against
`https://login.decisiv.net`, `Authorization: Bearer <JWT>`, `application/vnd.api+json`, every path scoped
by `{srm_account_id}`, **428 / `decisiv:access:003`** if the OAuth Application is not provisioned for
Service Management.

## Steps

1. **Find or create the customer.** `GET /service_management/{srm_account_id}/v1/customers` with
   `filter[name]` or `filter[name:like]` (never both — `decisiv:filters:012`). Create with
   `POST .../v1/customers`. If you operate across a service group, the parallel surface is
   `.../v1/service_group/customers`.
   - Contact data is validated hard: phone must be E.164 (`decisiv:phone:001`–`005` — leading `+`, no
     spaces, valid country code, 3–15 characters), email must satisfy RFC 3696 **and** resolve in DNS
     (`decisiv:email:001`–`004`), country must be ISO 3166-1 alpha-2 and state ISO 3166-2 with the two
     agreeing (`decisiv:location:000`–`004`).

2. **Find or create the customer asset.** `GET .../v1/customer_assets` filtering on
   `filter[vin]` / `filter[unit_number]` / `filter[chassis_id]`, or `POST .../v1/customer_assets`. At
   least one of `vin`, `chassis_id` or `unit_number` is required
   (`decisiv:registered_asset:010`). Review prior work with
   `GET .../v1/customer_assets/{id}/service_history`.
   - If the fleet has deactivated the asset you get **422** `decisiv:customer_assets:005` and cannot use
     it.

3. **Open the case — idempotently.**
   `POST /service_management/{srm_account_id}/v1/customer_assets/{id}/create_case`.

   **This is the one operation in the whole Decisiv surface with idempotency support. Use it.**
   Send `X-DECISIV-IDEMPOTENCY-KEY: <your key>` — max 128 characters
   (**400** `decisiv:idempotency_key:002` if longer). The first success for that (account, key) pair is
   cached briefly; a repeat replays the original response with `X-Decisiv-Idempotent-Replay: true`
   instead of creating a duplicate case. A concurrent request with the same in-flight key returns
   **409** `decisiv:idempotency_key:001` — that is "retry to receive the original response", not a
   failure. Generate the key from your own work-order id so a network timeout can never open two cases
   for one repair.

4. **Build the repair order.** `createCaseLineItem`
   (`POST .../v1/cases/{case_id}/line_items`), then `updateCaseLineItem` / `deleteCaseLineItem`.
   Underneath a line item you have parts, additional charges (and their categories), technician stories,
   internal labels and an assignee — each its own sub-resource under
   `.../v1/cases/{case_id}/line_items/{id}/...`.
   - Code the work with VMRS: the `.../v1/vmrs/...` reference endpoints serve components, work
     accomplished, reasons for repair, repair priorities, repair sites, positions, operator reports,
     asset types and technician part failure codes. At least one valid code is required (`vmrs-001`).
   - A closed case rejects line-item changes with **422** `decisiv:cases:line_items:001`.

5. **Record labour.** `POST .../v1/time_tasks` (plus GET/PATCH/DELETE on `{id}`) ties a technician's
   clocked time to the case; `.../v1/skill_levels` is the reference vocabulary.

6. **Sublet what you cannot do in-house.** `POST .../v1/sublet_requests` sends the work to another
   shop; `.../v1/sublet_associations` links the parent case to the sublet case. The rules are strict:
   both cases must be for the same asset (`decisiv:sublet_associations:004`), a case cannot be
   associated with itself (`005`), the parent case must have been created in the last 30 days (`002`),
   and only `Pending` sublet requests may be cancelled (`decisiv:sublet_requests:004`) while only
   `Canceled` ones may be reopened (`005`). Your account must participate in a configured Sublet
   Association Program or you get **412** `decisiv:sublet_associations:007`.

7. **Answer the customer.** `POST .../v1/customer_request/{id}/accept` or `.../decline`. Only requests in
   `Pending` may be answered — anything else is **428** `decisiv:customer_requests:001`.

8. **Close it.** `POST .../v1/cases/{case_id}/actions/close`. Closing a closed case is **409**
   `decisiv:cases:001`; opening an open case is **409** `decisiv:cases:002`. Both are idempotency
   signals in disguise — treat them as "already in the desired state", not as errors.

## Bulk work: silence your own echo

When you backfill or sync in bulk, send `X-DECISIV-SILENCE-EVENTS` listing the webhook events you do not
want fired back at you for that transaction (documented on 15 operations). An invalid value is **400**
`decisiv:silence_webhook_events:001`. Without it, a large import will fan out thousands of notifications
to your own endpoint.

## Rules that will bite you

- Errors are JSON:API `errors[]`, not RFC 9457. Branch on `code`, not `title`.
- **429 carries no rate-limit headers** — no numbers are published anywhere. Back off blind.
- `page[number]` / `page[size]` on every collection; `include` for related resources; `sort` for order.
- Attachments: multipart/form-data, 10 MB cap, extension must match the file.
- Metadata objects have key, value and total-size limits (`decisiv:metadata:001`–`004`).
