---
name: Pre-qualify a Sunbit customer
description: >-
  Send a customer a Sunbit pre-qualification link so they can see what they qualify for
  without a hard credit check, then react to the outcome webhook.
api: Sunbit Partner API
generated: '2026-07-31'
method: generated
source: https://docs.sunbit.com/docs/api-integrations/sunbit-pre-qualification
grounding: >-
  Sunbit publishes no OpenAPI, so `operations` below are the HTTP method + path exactly as
  published in Sunbit's documentation, not spec operationIds. Every path, header, field and
  error was transcribed from the cited docs pages; none is invented.
operations:
  - PUT /purchase/api/v1/online-link
  - GET /epay/api/v1/epay
events:
  - PREQUAL_COMPLETED
  - PREQUAL_APPROVED
  - PREQUAL_FAILED
  - PREQUAL_ABORTED
---

# Pre-qualify a Sunbit customer

Use this when a merchant wants to tell a customer, before any purchase, how much Sunbit will
finance for them. Pre-qualification does not affect the customer's credit score.

## Before you start

- Hosts: sandbox `https://api-sandbox.sunbit.com`, production `https://api.sunbit.com`.
- Auth: `sunbit-key` header. `sunbit-secret` is **not** required on this call.
- You need a `location` — the merchant identifier registered with Sunbit. It is the tenancy
  key for every call.
- Credentials come from the Sunbit Developers Portal (developers.sunbit.com). They are not
  self-serve; Sunbit must grant portal access first.

## Step 1 — request the pre-qualification link

`PUT /purchase/api/v1/online-link`

```
curl "https://api-sandbox.sunbit.com/purchase/api/v1/online-link" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json;charset=UTF-8' \
  -X PUT \
  -H "sunbit-key: YOUR_KEY" \
  --data-binary '[{"location":"myLocation","representativeEmail":"john@example.com","referral":"myReferral"}]'
```

Body fields:

| field | required | notes |
|---|---|---|
| `location` | yes | store id / name / any identifier of the location |
| `referral` | no | **set this** — it is the only thing that lets you match the webhook back to this request |
| `representativeEmail` / `representativeFirstName` / `representativeLastName` | no | the associate; Sunbit uses it to improve the associate experience |
| `customerPhoneNumber` | no | required if you want Sunbit to send the SMS |
| `customerDetails` | no | pre-fills the application for the customer |
| `sendSMS` | no | defaults to **true** — set `false` if you want to deliver the link yourself |
| `departmentId` | no | only send a value Sunbit gave you |
| `sourcePlatform` | no | the originating platform if different from the alliance |

Response is `{"url": "<sunbit-prequal-link>"}`. Display it or send it to the customer.

## Step 2 — handle the outcome webhook

Sunbit POSTs to your single configured webhook URL. Match on `payload.referral` (the
`referral` you sent):

```json
{
  "eventType": "PREQUAL_COMPLETED",
  "payload": {
    "purchaseId": "123",
    "location": "retailer",
    "approvalAmount": "1000.0",
    "purchaseAmountEntered": "500.0",
    "referral": "referral",
    "validUntil": "2021-09-01",
    "representativeEMail": "jason@email.com",
    "merchantFeeAmount": "5"
  }
}
```

`eventType` is one of `PREQUAL_COMPLETED`, `PREQUAL_APPROVED`, `PREQUAL_FAILED`,
`PREQUAL_ABORTED`.

**Verify the signature before trusting the body.** See
`skills/sunbit-verify-webhook-signature.md`.

Watch out for:

- `representativeEMail` in the payload — note the irregular capital `M`; it differs from the
  `representativeEmail` you sent.
- `approvalAmount` arrives as a **string**, not a number.
- `validUntil` is the offer expiry. The offer is not permanent.

## Step 3 — read the transaction later

Store `payload.purchaseId`. It is the key for lookup, void and refund. To read status:

`GET /epay/api/v1/epay?purchaseId=...` (or `?transactionId=...`) with `sunbit-key` and
optionally `sunbit-secret`. Status is `COMPLETED`, `INCOMPLETE` or `VOIDED`.

## Errors

| status | message | what to do |
|---|---|---|
| 403 | Bad credentials | key/secret wrong, or you are calling the wrong environment host |
| 404 | Not Found | the retailer for this alliance + location does not exist — confirm onboarding completed |
| 422 | Missing onlineApplication for retailer | retailer is not provisioned for the online application; contact Sunbit |
| 422 | Disabled onlineApplication for retailer | contact Sunbit |
| 500 | Internal Server Error | retry with backoff, then escalate to partnersupport@sunbit.com |

## Rules that apply to this whole API

- **No idempotency.** There is no `Idempotency-Key` header. `referral`/`transactionId` is a
  correlation key, not a dedupe key — replaying this call will mint another link. If a call
  times out, do not blindly retry; reconcile with `GET /epay/api/v1/epay` first.
- **US only.** Sunbit is unavailable in U.S. territories, and in WV for Home Services.
- **No rate-limit headers.** A per-key daily cap exists on checkout initialization but the
  value is unpublished and invisible at runtime.
