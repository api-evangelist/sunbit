---
name: Void or partially refund a Sunbit transaction
description: >-
  Cancel a completed Sunbit purchase in full (void) or return part of it (partial refund),
  and reconcile against the resulting webhook.
api: Sunbit Partner API
generated: '2026-07-31'
method: generated
source: https://docs.sunbit.com/docs/transactions-management/void-transactions
grounding: >-
  Sunbit publishes no OpenAPI, so `operations` are the HTTP method + path exactly as published
  in Sunbit's documentation, not spec operationIds. Transcribed from the cited docs pages.
operations:
  - GET /epay/api/v1/epay
  - PUT /epay/api/v1/epay/cancel/{purchaseId}
  - PUT /epay/api/v1/epay/changeAmount/{purchaseId}
events:
  - TRANSACTION_VOIDED
  - TRANSACTION_REFUNDED
---

# Void or partially refund a Sunbit transaction

Two different operations. Pick by whether the customer is getting **all** of the money back
or **some** of it.

| situation | operation |
|---|---|
| customer returned everything; order cannot be fulfilled at all | **void** — `PUT /epay/api/v1/epay/cancel/{purchaseId}` |
| customer returned some items; partial fulfilment; price correction | **partial refund** — `PUT /epay/api/v1/epay/changeAmount/{purchaseId}` |

Both require `sunbit-key` **and** `sunbit-secret`. Both act on a Sunbit-issued `purchaseId`.

## Step 0 — always read the transaction first

There is no idempotency on either operation and neither publishes an error table, so confirm
state before you mutate it.

```
curl 'https://api-sandbox.sunbit.com/epay/api/v1/epay?purchaseId=39325178' \
  -H 'sunbit-key: YOUR_KEY' -H 'sunbit-secret: YOUR_SECRET'
```

Proceed only if `status` is `COMPLETED`. If it is already `VOIDED`, stop — you are done. If
`INCOMPLETE`, there is nothing to reverse.

## Void — full cancellation

`PUT /epay/api/v1/epay/cancel/{purchaseId}`

```
curl -X PUT "https://api-sandbox.sunbit.com/epay/api/v1/epay/cancel/39325178" \
  -H "accept: */*" \
  -H "sunbit-key: YOUR_KEY" \
  -H "sunbit-secret: YOUR_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"transactionId":"23259d52f67fb8","location":"retailer4"}'
```

Body: `transactionId` (yours, same one used at initialization) and `location` — both
required.

Response: `{"purchaseId":"39325178","status":"VOIDED"}`

This cancels the **entire** purchase immediately, issues a full refund, and notifies both the
customer and the merchant. It is not reversible through the API.

## Partial refund

`PUT /epay/api/v1/epay/changeAmount/{purchaseId}`

```
curl -X PUT 'https://api-sandbox.sunbit.com/epay/api/v1/epay/changeAmount/39325178' \
  -H 'Content-Type: application/json' \
  -H 'accept: */*' \
  -H 'sunbit-key: YOUR_KEY' \
  -H 'sunbit-secret: YOUR_SECRET' \
  -d '{"transactionId":"23259d52f67fb8","location":"retailer4","reason":"PARTIAL_REFUND","returnedAmount":110}'
```

| field | required | notes |
|---|---|---|
| `transactionId` | yes | your internal id from initialization |
| `location` | yes | |
| `reason` | yes | `PARTIAL_REFUND` (customer returned items) or `ERROR_CORRECTION` (fulfilment issue or price adjustment) |
| `returnedAmount` | yes | decimal, **must be less than** the original total |

Response: `{"purchaseId":"39325178","changedAmount":490}` — `changedAmount` is the *net*
purchase amount remaining, not the amount refunded.

If `returnedAmount` equals the full total, use **void** instead.

## Reconcile on the webhook

Both operations emit an event to your configured webhook URL:

```json
{
  "eventType": "TRANSACTION_REFUNDED",
  "payload": {
    "purchaseId": "938",
    "location": "retailer",
    "purchaseAmount": "140.0",
    "netPurchaseAmount": "139.0",
    "purchaseDate": "2022-04-13 23:27:06",
    "modificationDate": "2022-04-20 01:42:51",
    "referral": "123881",
    "advisorName": null,
    "representativeName": null,
    "merchantFeeAmount": 5
  }
}
```

- `eventType` is `TRANSACTION_VOIDED` or `TRANSACTION_REFUNDED`.
- `purchaseAmount` is the **original** total; `netPurchaseAmount` is what remains — `0` for a
  void, the adjusted amount for a refund.
- These events also fire for voids/refunds performed **outside** the API (e.g. in the Sunbit
  merchant tooling), so treat the webhook — not your own API call — as the system of record.
- Verify the `Sunbit-Signature` before trusting the body.

## Cautions

- **No idempotency and no documented errors.** These are the only two money-moving operations
  in the API with no published error table. Read state first, act once, and reconcile from the
  webhook rather than retrying.
- `advisorName` is deprecated in favour of `representativeName`.
- Sandbox: prefix your `transactionId` with `VOIDED_` to simulate an already-voided
  transaction on the lookup call.
