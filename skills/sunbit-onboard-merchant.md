---
name: Onboard a merchant location to Sunbit
description: >-
  Sign a merchant on your platform up for Sunbit — create the location, track the application
  through its status webhook, and poll the status endpoint.
api: Sunbit Partner API
generated: '2026-07-31'
method: generated
source: https://docs.sunbit.com/docs/onboarding/adding-new-location
grounding: >-
  Sunbit publishes no OpenAPI, so `operations` are the HTTP method + path exactly as published
  in Sunbit's documentation, not spec operationIds. Verticals, states and error rows are
  transcribed verbatim from the cited page.
operations:
  - POST /onboarding/api/v1/location
  - GET /onboarding/api/v1/location/{location}
events:
  - MERCHANT_CREATED
  - MERCHANT_LOCATION_DETAILS_ADDED
  - MERCHANT_CONTACT_DETAILS_ADDED
  - MERCHANT_BANK_INFORMATION_ADDED
  - MERCHANT_LEGAL_INFORMATION_ADDED
  - MERCHANT_SUBMITTED
  - MERCHANT_ACTIVATED
  - MERCHANT_DECLINED
---

# Onboard a merchant location to Sunbit

For platforms that want their own merchants to be able to offer Sunbit. You either ask Sunbit
to text the merchant an onboarding link, or you take back a URL and embed it in your own
signup flow.

Both need `sunbit-key` **and** `sunbit-secret`.

## Step 1 — create the location

`POST /onboarding/api/v1/location`

```
curl -X POST 'https://api-sandbox.sunbit.com/onboarding/api/v1/location' \
  -H 'Content-Type: application/json' \
  -H 'sunbit-key: YOUR_KEY' \
  -H 'sunbit-secret: YOUR_SECRET' \
  -d '{
    "location": "johns-store",
    "sendSMS": true,
    "legalName": "John Doe Dental",
    "vertical": "Dental",
    "representativeDetails": {
      "firstName": "John", "lastName": "Doe",
      "phoneNumber": "303-988-8945", "email": "john.doe@example.com"
    }
  }'
```

| field | required | notes |
|---|---|---|
| `location` | yes | **you** choose it — the unique identifier for this merchant, and the tenancy key on every later call. Choose carefully; it cannot be re-created. |
| `vertical` | yes | must be one of the 14 values below |
| `representativeDetails` | yes | the merchant contact |
| `sendSMS` | no | defaults **false**. If true, `representativeDetails.phoneNumber` is required or you get a 400. |
| `legalName`, `businessName`, `businessAddress`, `sourcePlatform` | no | anything you supply pre-fills the merchant's form |

Response:
```json
{"location":"johns-store","url":"https://merchant-onboarding.sunbit.com/hwerityQas","creationDate":"2019-12-19","status":"CREATED"}
```

The `url` **expires in 30 days** if the application is not submitted.

### Verticals (closed enum)

`Car_Dealerships` `Car_Services` `Veterinary` `Motorsports_Parts_and_Services` `Dental`
`Eyewear` `Home_Services` `Med_Spa` `Bridal` `General_Retail` `Jewelry_and_Watches`
`Legal_Services` `Medical_Office` `Hospital_Health_System`

Anything else returns 422.

### Eligibility to check before you call

- Sunbit is **US only** and is unavailable in U.S. territories.
- **West Virginia** is not supported for the `Home_Services` vertical.
- Business information for **Dental and Healthcare** merchants in **CA, NY and IL** will not
  be pre-filled — send it anyway, but expect the merchant to re-enter it.

## Step 2 — track the application

Two ways, and you should use both.

**Webhook** (push) — fires on every state change to your configured URL:

```json
{"eventType":"MERCHANT_CREATED","payload":{"location":"retailer","url":"merchant/application/url","statusReason":"NONE"}}
```

Sequence: `MERCHANT_CREATED` → `MERCHANT_LOCATION_DETAILS_ADDED` →
`MERCHANT_CONTACT_DETAILS_ADDED` → `MERCHANT_BANK_INFORMATION_ADDED` →
`MERCHANT_LEGAL_INFORMATION_ADDED` → `MERCHANT_SUBMITTED` → `MERCHANT_ACTIVATED`, or
`MERCHANT_DECLINED` at any point. On decline, `statusReason` carries a human-readable reason;
otherwise it is the literal string `"NONE"`.

**Polling** (pull) — `GET /onboarding/api/v1/location/{location}`:

```
curl 'https://api-sandbox.sunbit.com/onboarding/api/v1/location/johns-store' \
  -H 'sunbit-key: YOUR_KEY' -H 'sunbit-secret: YOUR_SECRET'
```

```json
{"location":"johns-store","url":"https://merchant-onboarding.sunbit.com/hwerityQas","creationDate":"2019-12-19","status":"LOCATION_DETAILS_ADDED","expiredInDays":30,"expirationDate":"2020-01-19"}
```

Status enum here drops the `MERCHANT_` prefix used by the webhooks: `CREATED`,
`LOCATION_DETAILS_ADDED`, `CONTACT_DETAILS_ADDED`, `BANK_INFORMATION_ADDED`, `SUBMITTED`,
`LEGAL_INFORMATION_ADDED`, `ACTIVATED`, `DECLINED`. Map between the two forms.

Use `expiredInDays` / `expirationDate` to nudge merchants before the link dies.

## Step 3 — after activation

Only once the location reaches `ACTIVATED` can you use it in pre-qualification, estimation,
text-to-pay or checkout calls. Sunbit sends the merchant a training package directly.

## Errors

| status | message | action |
|---|---|---|
| 400 | Bad Request | missing/invalid input, or `sendSMS: true` with no representative phone number |
| 401 | Unauthorized | key/secret do not match any alliance — note this service uses **401**, while /epay and /purchase use 403 for the same failure |
| 409 | Conflict | this location already exists for your alliance |
| 422 | Vertical parameter is not supported | use one of the 14 values |
| 422 | State is not supported | unsupported state for this vertical |
| 500 | Internal Server Error | retry with backoff |

Status lookup adds: 404 `Resource Not Found` (unknown location), 422 `State Not Supported`.

## Safety rules

- **Onboarding is not idempotent, and it fails loudly rather than quietly.** A repeat POST for
  the same `location` returns **409 Conflict**, not a no-op success. On any timeout or unknown
  outcome, call `GET /onboarding/api/v1/location/{location}` to find out whether the create
  landed — never blind-retry the POST.
- `location` is chosen by you and is permanent. Use a stable internal identifier, not a
  display name that a merchant might rename.
