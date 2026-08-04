# Sunbit

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

Sunbit is a Los Angeles based financial technology company providing point-of-sale buy now,
pay-over-time financing for everyday needs — auto service and dealerships, dental, eyewear,
veterinary, med spa, home services and general retail. Merchants and SaaS platforms integrate
through a partner REST API and a hosted JavaScript SDK.

- Website — https://sunbit.com/
- Developer docs — https://docs.sunbit.com/docs/overview/overview
- Developers portal — https://developers.sunbit.com/
- Status — https://status.sunbit.com/
- Integration partners — https://sunbit.com/integration-partners/

## API surface

| Host | Environment |
|---|---|
| `https://api.sunbit.com` | production |
| `https://api-sandbox.sunbit.com` | sandbox |

Authentication is a `sunbit-key` / `sunbit-secret` header pair, with IP allowlisting on
checkout initialization and the customer-offer-history report. There is no OAuth, no OIDC and
no scope model.

Fourteen documented operations across six services — `purchase` (pre-qualification,
estimation), `epay` (checkout, text-to-pay, lookup, void, refund), `onboarding` (merchant
locations), `reports`, `alliance` (Payment Path auth) and `developers-portal-service`
(embedded dashboard) — plus 20 webhook event types signed with HMAC-SHA256.

**Sunbit publishes no machine-readable contract.** No OpenAPI, Swagger, GraphQL, AsyncAPI,
gRPC/proto, MCP server or A2A agent card was found on any of the eight Sunbit hosts probed.
Every artifact in this repository is therefore transcribed or derived from Sunbit's published
prose documentation, with the source URL recorded on each entry.

## Artifacts

| Path | What it holds |
|---|---|
| `authentication/` | credential schemes, bearer variants, IP allowlisting, credential issuance |
| `conventions/` | pagination, versioning, error envelopes, sentinel values, rate limiting, idempotency (absent) |
| `errors/` | 44 documented errors across 12 operations |
| `asyncapi/` | the 20-event webhook catalog and signature scheme (no AsyncAPI is published) |
| `sandbox/` | sandbox hosts, SDK modes, transactionId simulation prefixes, demo test values |
| `data-model/` | entities and relationships derived from the documented field tables |
| `components/` | the hosted browser SDK surface |
| `packages/` | client library inventory across eight registries |
| `lifecycle/` | versioning, deprecation notices, status page, availability constraints |
| `conformance/` | 34 standards assertions with evidence |
| `security/` | domain security probe, vulnerability disclosure |
| `well-known/` | `/.well-known` probe results across all eight hosts |
| `skills/` | five packaged agent skills for the marquee flows |
| `llms/` | generated `llms.txt` |
