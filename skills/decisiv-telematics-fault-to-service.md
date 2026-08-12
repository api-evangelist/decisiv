---
name: Push a telematics fault into SRM and turn it into service
description: >-
  As a telematics provider or OEM, post diagnostic readings and fault codes for a registered asset into
  the Decisiv SRM Gateway, then read them back on the case they inform.
api: openapi/decisiv-telematics-openapi.yml
base_url: https://srm-api.decisivapps.com
operations:
  - createDiagnosticReading
  - listDiagnosticReadings
  - getDiagnosticReading
  - createFault
  - listFaults
  - getFault
paths:
  - GET /service_management/{srm_account_id}/v1/cases/{case_id}/faults
  - GET /service_management/{srm_account_id}/v1/cases/{case_id}/diagnostic_readings
generated: '2026-08-12'
method: generated
---

# Telematics fault → service event (Decisiv SRM Gateway)

The Telematics module is small — 4 paths, 6 operations — but it is the entry point for connected-asset
data into the SRM Ecosystem. Grounded in `openapi/decisiv-telematics-openapi.yml`.

## Before you start

OAuth 2.0 authorization code against `https://login.decisiv.net`; `Authorization: Bearer <JWT>`;
`application/vnd.api+json`; paths scoped by `{srm_account_id}`. Your OAuth Application must be
provisioned for the **Telematics** module specifically — provisioning is per module, so an application
that can read cases may still get **428** `decisiv:access:003` here.

## Steps

1. **Post the diagnostic reading.** `createDiagnosticReading`
   (`POST /telematics/{srm_account_id}/v1/diagnostic_readings`). A reading carries the asset it belongs
   to and a `source_account` — the account that supplied it. Readings are addressed by `uuid`, not an
   integer id.

2. **Post the fault.** `createFault` (`POST /telematics/{srm_account_id}/v1/faults`). A fault relates to
   an asset and to the diagnostic reading it came from, so post the reading first and reference it.

3. **Read them back.** `listDiagnosticReadings` / `listFaults` with `filter[...]` and
   `page[number]` / `page[size]`; `getDiagnosticReading` / `getFault` by `uuid`.

4. **See them on the case.** Once a service event exists, the Service Management module exposes the same
   data in case context at
   `GET /service_management/{srm_account_id}/v1/cases/{case_id}/faults` and
   `GET /service_management/{srm_account_id}/v1/cases/{case_id}/diagnostic_readings`. That is the join
   that makes this worth doing: the technician sees the fault that brought the truck in.

5. **Keep the meter current.** Meter readings live on the asset, not the fault:
   `asset_management_ra_update_meter_data`
   (`POST /asset_management/{srm_account_id}/v1/registered_assets/{id}/update_meter_data`) in the Asset
   Management module. An `odometer_unit` is mandatory whenever you send an odometer value
   (`decisiv:registered_asset:011`) and must be one of the supported units
   (`decisiv:registered_asset:012`). Meter values must be integers and the unit of measure must be
   supported (`decisiv:meter_data:001`–`003`).

## Rules that will bite you

- **Identify the asset correctly.** VIN must be 17 characters, valid check digit, no `I`/`O`/`Q`
  (`decisiv:vins:001`–`006`). A serial number must match the last 8–9 digits of the VIN
  (`decisiv:vin_based_asset:002`). Getting this wrong silently attaches telematics to the wrong truck —
  or, more often, gets rejected at 422.
- **Deactivated assets are inert.** **409** `decisiv:registered_asset:009`.
- **Timestamps are ISO 8601 `YYYY-MM-DDTHH:MM:SSZ`** (`decisiv:request_attributes:006`) and may not be
  in the future (`decisiv:request_attributes:012`). Telematics feeds with clock skew will be rejected.
- **429 has no headers.** Telematics is the highest-volume surface here and the one most likely to be
  throttled, and Decisiv publishes no limit, no window and no `Retry-After`. Batch, pace yourself, and
  back off exponentially on 429.
- **424 Failed Dependency** means the platform cannot resolve a dependency for the asset — not
  retryable by you.
