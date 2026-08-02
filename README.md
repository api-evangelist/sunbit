# Sunbit

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
