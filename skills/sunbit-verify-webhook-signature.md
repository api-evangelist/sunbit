---
name: Verify a Sunbit webhook signature
description: >-
  Authenticate an inbound Sunbit webhook by recomputing its HMAC-SHA256 signature and
  rejecting stale or forged deliveries.
api: Sunbit Partner API
generated: '2026-07-31'
method: generated
source: https://docs.sunbit.com/docs/webhooks/webhooks-verify-signature
grounding: >-
  Transcribed from Sunbit's published Verify Webhook Signature guide. Sunbit publishes no
  OpenAPI and no AsyncAPI; this skill covers the inbound HTTP endpoint you host.
operations:
  - POST <your-webhook-url>
---

# Verify a Sunbit webhook signature

Every Sunbit webhook carries a `Sunbit-Signature` header. Verify it before you parse, trust,
or act on the body — the endpoint is public and otherwise unauthenticated.

## The header

```
Sunbit-Signature: t=1565220904,v1=20c75c1180c701ee8a796e81507cfd5c932fc17cf63a4a55566fd38da3a2d3d2
```

- `t` — unix timestamp in **seconds**.
- `v1` — hex HMAC-SHA256. `v1` is currently the only valid scheme; schemes start with `v`
  followed by an integer, so parse defensively and ignore unknown schemes rather than failing.

## Getting the secret

The signing secret is generated automatically when you set your webhook endpoint in the
Sunbit Developers Portal, under the Webhooks section. It is per-endpoint and per-environment
(sandbox and production have different secrets). Rotation is not documented.

## The four steps

**1. Split the header.** Separate on `,`, then each element on `=`, giving `t` and `v1`.

**2. Build the signed payload.** Concatenate:

```
<timestamp> + "." + <raw request body>
```

Use the **raw bytes as received**. Do not re-serialize parsed JSON — key order and whitespace
change the hash.

**3. Compute the expected signature.** HMAC-SHA256 over that string, keyed with your webhook
secret, hex-encoded.

**4. Compare, and check freshness.** Compare against `v1`. Then compute the difference between
now and `t` and reject anything outside your tolerance — Sunbit recommends up to **5 minutes**.
Without the freshness check, a captured delivery can be replayed forever.

Use a constant-time comparison. The published example uses `===`; prefer
`crypto.timingSafeEqual` or your language's equivalent.

## Reference implementation (from Sunbit's docs)

```js
const crypto = require('crypto');

const headerRegex = /t=(?<timestamp>\d{10}),v1=(?<signature>[a-f0-9]+)/;
const secret = 'DwS3QStMkgKziZxd9NXcvqFkxP4JNA3i';

const header = 't=1643444288,v1=e1bfa98d067faeea521387c8917b71c96e32e1f9028a3b0b2167c4c7408cdacb';
const found = header.match(headerRegex);
const timestamp = found.groups.timestamp;

const payload = timestamp + "." + JSON.stringify({
  "eventType": "MERCHANT_CREATED",
  "payload": { "location": "Merchant location", "url": "merchant/application/url", "statusReason": "NONE" }
});

const hmac = crypto.createHmac('sha256', secret);
const gen_hmac = hmac.update(payload, 'utf8').digest('hex');

if (gen_hmac === found.groups.signature) console.log('Valid signature');
else console.log('Invalid signature');
```

The secret above is Sunbit's published worked example, not a live credential.

Note the docs' regex hard-codes a 10-digit timestamp — that breaks in the year 2286 and, more
practically, on any change in timestamp width. Prefer `\d+`.

## Testing it

1. Point your Webhook URL in the Developers Portal at `webhook.site` or an ngrok tunnel.
2. Fire a delivery with the portal's test button, or by calling the pre-qualification or
   merchant onboarding APIs.
3. Copy the `sunbit-signature` header and the payload from the receiver.
4. Run them through your verifier.

## What the signature does not give you

- **No event id.** The envelope is only `{eventType, payload}` — there is no delivery id or
  event id, so deduplicate on `payload.purchaseId` + `eventType` (or `payload.location` +
  `eventType` for onboarding events).
- **No retry or ordering guarantees** are documented. Assume events can arrive out of order
  and more than once, and make your handler idempotent on your side.
- **One URL per environment.** There is no per-event subscription or filtering, so your single
  endpoint receives all 20 event types across all five event groups; branch on `eventType`.

## Event types you may receive

`MERCHANT_CREATED` `MERCHANT_LOCATION_DETAILS_ADDED` `MERCHANT_CONTACT_DETAILS_ADDED`
`MERCHANT_BANK_INFORMATION_ADDED` `MERCHANT_LEGAL_INFORMATION_ADDED` `MERCHANT_SUBMITTED`
`MERCHANT_ACTIVATED` `MERCHANT_DECLINED` `PREQUAL_COMPLETED` `PREQUAL_APPROVED`
`PREQUAL_FAILED` `PREQUAL_ABORTED` `TEXT_TO_PAY_COMPLETED` `TEXT_TO_PAY_FAILED`
`TEXT_TO_PAY_ABORTED` `CHECKOUT_SDK_COMPLETED` `CHECKOUT_SDK_FAILED` `CHECKOUT_SDK_ABORTED`
`TRANSACTION_VOIDED` `TRANSACTION_REFUNDED`
