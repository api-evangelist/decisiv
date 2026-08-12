# Decisiv

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Decisiv operates the Service Relationship Management (SRM) platform for the commercial vehicle industry —
the network where fleets, asset owners, OEMs, dealers, independent service providers and component
suppliers work a single service event together. The APIs cover asset registration and VIN identification,
service requests, cases, estimates, line items with VMRS coding, parts, labour, sublet work between shops,
and telematics-driven faults and diagnostic readings.

## APIs profiled here

| API | Spec | Ops | Base |
|---|---|---|---|
| SRM Gateway — Account Management | OpenAPI 3.1.0 | 21 | `https://srm-api.decisivapps.com` |
| SRM Gateway — Asset Management | OpenAPI 3.1.0 | 85 | `https://srm-api.decisivapps.com` |
| SRM Gateway — Service Management | OpenAPI 3.1.0 | 139 | `https://srm-api.decisivapps.com` |
| SRM Gateway — Telematics | OpenAPI 3.1.0 | 6 | `https://srm-api.decisivapps.com` |
| Global Assets API | Swagger 2.0 (v2.1.5) | 10 | `https://global-assets.decisivapps.com/api/v1/` |
| Service Provider API | Swagger 2.0 (v2.0.4) | 11 | `https://service-provider.decisivapps.com/api/v1/` |
| Platform API (legacy XML) | XSD bundle | — | `https://api.decisiv.net/platform_api` |

All specs were harvested verbatim from Decisiv's own public documentation hosts on 2026-08-12 and are in
`openapi/_original/`.

## What Decisiv publishes well

- **Six machine-readable specs, 272 operations**, all JSON:API (`application/vnd.api+json`).
- **A 112-code namespaced error registry** (`decisiv:<domain>:<nnn>`) published inside the spec examples —
  VIN check digits, E.164 phone rules, RFC 3696 email rules, filter grammar, sublet eligibility.
- **A managed webhook surface** with 18 named events, rotatable signing keys and an account-scoped event
  catalog endpoint — plus `X-DECISIV-SILENCE-EVENTS` to mute your own echo during bulk writes.
- **OAuth 2.0 with real discovery metadata** at `login.decisiv.net` (RFC 8414 + OIDC, PKCE, ES256 JWKS).
- **ISO/IEC 27001:2022 certification** since February 2024, certificate downloadable from the Trust Center.

## Where the gaps are

- **No API client SDK in any language.** The only first-party packages are the `@decisiv/*` Key Design
  System React components on npm, last released **August 2020**.
- **No changelog for the current product.** The only dated changelog covers the legacy Platform API and
  stops at 2021-09-30.
- **No status page, no deprecation policy, no Sunset headers** — even though the OAuth password grant is
  marked deprecated in the specs with no removal date.
- **No published rate limits.** 429 is declared on 37 operations with no number, no window and no
  `RateLimit-*`/`Retry-After` header.
- **No pricing, no self-serve signup.** Access is provisioned per OAuth Application per module by sales.
- **No MCP server, no A2A agent card, no AsyncAPI, no security.txt.**
- 118 of 272 operations carry no `operationId`, almost all of them in Service Management.

Links:
- https://www.decisiv.com/
- https://api-docs.decisiv.net/
- https://srm-api.decisivapps.com/api-docs/v1
- https://www.decisivmarketplace.com/solutions-center/
- https://forgeglobal.com/decisiv_stock/
