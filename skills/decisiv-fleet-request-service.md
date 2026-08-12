---
name: Request service on a fleet asset and respond to the estimate
description: >-
  As a fleet or asset manager, find a registered asset, pick a service provider, raise a service request,
  follow the case it opens, and approve or decline the estimate that comes back — using the Decisiv SRM
  Gateway Asset Management API.
api: openapi/decisiv-asset-management-openapi.yml
base_url: https://srm-api.decisivapps.com
operations:
  - listRegisteredAssets
  - getRegisteredAsset
  - listServiceProviders
  - createServiceRequest
  - getServiceRequest
  - cancelServiceRequest
  - listCases
  - getCase
  - createCaseNote
  - sendNoteToProvider
  - listEstimates
  - getEstimate
  - respondToEstimate
generated: '2026-08-12'
method: generated
---

# Request service on a fleet asset (Decisiv SRM Gateway)

This is the fleet / asset-owner side of a service event. Everything below is grounded in operations that
exist verbatim in `openapi/decisiv-asset-management-openapi.yml`.

## Before you start

- **Auth.** OAuth 2.0 authorization code against `https://login.decisiv.net`
  (`/auth/api_gateway` → `/oauth/token`, PKCE `S256` supported). Send `Authorization: Bearer <JWT>`.
  The password grant still works but is marked **deprecated** in the spec — do not build on it.
- **Provisioning.** Your OAuth Application must be provisioned for the Asset Management module. If it is
  not, every call returns **428** with code `decisiv:access:003`. That is a configuration problem you
  cannot fix from the client; contact Decisiv.
- **Account scope.** Every path is prefixed with `{srm_account_id}`. You are always acting inside one
  SRM account.
- **Media type.** Send and accept `application/vnd.api+json`. Bodies are JSON:API — `data.type`,
  `data.attributes`, `data.relationships`.

## Steps

1. **Find the asset.** `listRegisteredAssets`
   (`GET /asset_management/{srm_account_id}/v1/registered_assets`). Filter with `filter[vin]`,
   `filter[unit_number]`, `filter[chassis_id]` or `filter[serial_number]`. Filter values must be at
   least 3 and fewer than 4096 characters (`decisiv:filters:007` / `decisiv:filters:008`), and a filter
   cannot be combined with its own `:like` variant (`decisiv:filters:012`).
   Confirm with `getRegisteredAsset` if you need the full record; `getBuildInformation` returns OEM
   build detail where authorized.
   - A VIN you submit must be 17 characters with a valid check digit and no `I`, `O` or `Q`
     (`decisiv:vins:001`–`006`). Validate before you send.
   - If the asset is deactivated you get **409** `decisiv:registered_asset:009`; it cannot be used.

2. **Pick the service provider.** `listServiceProviders`
   (`GET .../v1/service_providers`), then `getServiceProvider` for detail. If the provider has no
   contact configured to receive requests, step 3 fails with **422**
   `decisiv:service_requests:002` — pick another provider rather than retrying.

3. **Raise the request.** `createServiceRequest` (`POST .../v1/service_requests`) with relationships to
   the registered asset, the service provider and a primary contact. If the asset is not enrolled with
   any dealer on the account you get **422** `decisiv:service_requests:001`.
   - This operation is **not** idempotent. There is no idempotency key on the fleet side — the only
     idempotent write in the whole Gateway is service-provider case creation. Track your own
     `external_reference` and check `listServiceRequests` before retrying a request whose response you
     did not see.

4. **Watch it become a case.** Poll `listCases` / `getCase`, or better, subscribe to
   `decisiv:asset_management:service_request:accepted`,
   `decisiv:asset_management:service_request:declined` and `decisiv:asset_management:case:created`
   (see `skills/decisiv-subscribe-to-service-events.md`). Use `listCaseSummaries` for a light list view.
   To cancel before acceptance, `cancelServiceRequest` (`POST .../v1/service_requests/{id}/cancel`).

5. **Talk to the shop.** `createCaseNote` (`POST .../v1/cases/{case_id}/notes`) records a note on the
   case; `sendNoteToProvider` (`POST .../v1/cases/{case_id}/send_note_to_provider`) pushes it to the
   service provider. Attach documents with `createCaseAttachment` — multipart/form-data, **10 MB
   maximum** (`decisiv:attachments:002`), and the filename extension must match the real file
   (`decisiv:attachments:003`).

6. **Approve or decline the estimate.** `listEstimates` / `getEstimate`, then `respondToEstimate`
   (`POST .../v1/estimates/{id}/respond`) carrying the decision. Estimates are versioned, and the
   approval decision may carry a reason and reason description. Subscribe to
   `decisiv:asset_management:estimate:approval_requested` so you are not polling for the moment the
   estimate lands.

7. **Keep the meter honest.** `asset_management_ra_update_meter_data`
   (`POST .../v1/registered_assets/{id}/update_meter_data`) posts an odometer/hour reading;
   `asset_management_ra_meter_data_history` reads it back. An odometer reading requires an
   `odometer_unit` (`decisiv:registered_asset:011`) drawn from the supported values
   (`decisiv:registered_asset:012`).

## Rules that will bite you

- **Errors are JSON:API, not RFC 9457.** Parse `errors[]` with `{title, detail, code, status, source}`.
  Branch on the namespaced `code` (`decisiv:<domain>:<nnn>`), never on `title` — see
  `errors/decisiv-error-codes.yml` for all 112 published codes.
- **428 and 424 are not your fault and not retryable.** 428 means the OAuth Application is not
  provisioned for the module. 424 means an unresolved data dependency inside the Decisiv platform.
  Retrying either is wasted budget; escalate to Decisiv.
- **429 has no signal.** Decisiv publishes no rate-limit numbers and returns no `RateLimit-*` or
  `Retry-After` header. On 429, back off exponentially with jitter starting at several seconds and log
  it — you will not be told when the window resets.
- **Paginate everything.** Collections use `page[number]` and `page[size]`; totals live in `meta`.
- **Timestamps are ISO 8601 `YYYY-MM-DDTHH:MM:SSZ`** (`decisiv:request_attributes:006`), and some date
  range filters are capped at 180 days (`decisiv:filters:004`).
