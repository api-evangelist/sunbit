---
name: Take a Sunbit checkout payment
description: >-
  Add Sunbit as a payment method: initialize a transaction server-side for a JWT, open the
  hosted checkout modal in the browser, then validate the result.
api: Sunbit Partner API
generated: '2026-07-31'
method: generated
source: https://docs.sunbit.com/docs/sdk-integrations/sunbit-checkout-sdk
grounding: >-
  Sunbit publishes no OpenAPI, so `operations` are the HTTP method + path exactly as published
  in Sunbit's documentation, not spec operationIds. Every path, header, field, enum and error
  is transcribed from the cited docs page.
operations:
  - POST /epay/api/v1/epay
  - GET /epay/api/v1/epay
sdk:
  - SUNBIT.init
  - SUNBIT.UI.checkoutElement
  - SUNBIT.epay.init
  - SUNBIT.epay.checkout
events:
  - CHECKOUT_SDK_COMPLETED
  - CHECKOUT_SDK_FAILED
  - CHECKOUT_SDK_ABORTED
---

# Take a Sunbit checkout payment

Five moving parts: your server initializes and gets a token, your page renders a button, the
customer completes Sunbit's hosted modal, your callbacks fire, and your server validates.

**The secret never touches the browser.** `sunbit-secret` is server-side only; the browser
gets the publishable `sunbitKey` and a per-transaction JWT.

## Step 1 — server: initialize the transaction

`POST /epay/api/v1/epay` — requires **both** `sunbit-key` and `sunbit-secret`, and
`Content-Type: application/json` (nothing else is accepted).

```
curl -X POST 'https://api-sandbox.sunbit.com/epay/api/v1/epay' \
  -H 'Content-Type: application/json' \
  -H 'sunbit-key: YOUR_KEY' \
  -H 'sunbit-secret: YOUR_SECRET' \
  -d '{
    "transactionId": "000000931",
    "amount": 1008,
    "location": "Location1",
    "representativeEmail": "jason@email.com",
    "customerDetails": { "firstName": "John", "lastName": "Doe", "email": "john.doe@example.com", "phone": "3039888945" },
    "isShippingToStore": false,
    "items": [{ "serialNumber": "1111111", "amount": 166 }]
  }'
```

Required: `transactionId` (yours), `amount` (up to 2 decimals), `location`.
Optional: `referenceNumber`, `departmentName`, `customerDetails`, `shippingAddress`,
`isShippingToStore`, `items[]`, `representativeEmail`.

Response: `{"token": "<JWT>"}`. Return only that token to your page.

Note `shippingAddress` uses `street1`/`zipCode` while `customerDetails.addressDetails` uses
`address`/`zipcode` — the two address objects are not the same shape.

## Step 2 — browser: load the SDK and render the button

```html
<script>
  window.sunbitAsyncInit = function () {
    SUNBIT.init({ sunbitKey: '<PUBLISHABLE KEY>', mode: 'SANDBOX' });
    SUNBIT.UI.checkoutElement('[data-sunbit-checkout]', { theme: 'transparent', size: 'large' });
  };
</script>
<script async defer src="https://static.sunbit.com/sdk/sunbit-sdk.js"></script>
```

`size`: `small` | `medium` (default) | `large`. `theme`: `dark` (default) | `light` | `white`
| `transparent`.

## Step 3 — browser: open the modal

Attach the click listener **inside** `SUNBIT.epay.init`'s `onInitFinish` — earlier and it
will not fire.

```js
SUNBIT.epay.init({
  onInitFinish: () => {
    button.addEventListener('click', () => {
      SUNBIT.epay.checkout({
        token: data.sunbitToken,
        onCompleted:       d => handleCompleted(d.statusType, d.purchaseId),
        onCanceled:        d => handleCancellation(d.statusType),
        onUserCancelled:   d => handleUserCancellation(d.statusType),
        onPlaceInFlow:     d => handlePage(d.page),
        onPurchaseVerified:d => handleVerified(d.purchaseId),
      });
    });
  }
});
```

`onPlaceInFlow` reports the customer's position through `PHONE_NUMBER`,
`PHONE_VERIFICATION`, `CUSTOMER_DETAILS`, `EMAIL_VERIFICATION`, `PAYMENT_PLAN`,
`PAYMENT_METHOD`, `AGREEMENT`, `AUTHORIZATION`, `THANK_YOU`. The enum **can repeat** in one
flow — only the last callback is authoritative.

## Step 4 — server: validate

Never fulfil on the browser callback alone.

```
curl 'https://api-sandbox.sunbit.com/epay/api/v1/epay?transactionId=...&purchaseId=39325178' \
  -H 'sunbit-key: YOUR_KEY' -H 'sunbit-secret: YOUR_SECRET'
```

```json
{"purchaseId":"mockId","purchaseAmount":100,"status":"COMPLETED","purchaseDate":"2020-06-16T01:02:42"}
```

`status` is `COMPLETED` (fulfil), `INCOMPLETE` (still in progress — `purchaseId`,
`purchaseAmount` and `purchaseDate` may be null) or `VOIDED`.

You will also receive a `CHECKOUT_SDK_COMPLETED` / `_FAILED` / `_ABORTED` webhook. Note the
webhook types `purchaseAmount` and `merchantFeeAmount` as **strings** here, while the
otherwise identical `TEXT_TO_PAY_*` events type them as numbers.

## Step 5 — sandbox before production

Force any outcome by prefixing your `transactionId`:

`COMPLETED_` `INCOMPLETE_` `VOIDED_` `BAD_CREDENTIALS_` `IP_NOT_ALLOWED_`
`LOCATION_NOT_EXIST_` `DAILY_LIMIT_` `LOW_AMOUNT_` `INTERNAL_` — e.g.
`COMPLETED_3423lgregr`. On the validation call also `TRANSACTION_NOT_ALLOWED_`,
`PURCHASE_NOT_ALLOWED_`, `WRONG_PURCHASE_`.

Default SDK mode `SANDBOX` never calls Sunbit's servers and yields no real `purchaseId`.
Switch to `mode: 'demo'` for a real one — demo pin is `1234`, phone `212.555.9999` simulates
an email-verified customer, and the email must be one you can actually verify.

Going live: swap host to `https://api.sunbit.com`, swap key/secret/location,
**strip every simulation prefix**, and set `mode: 'prod'`.

## Errors

| status | message | action |
|---|---|---|
| 403 | Bad credentials | wrong key/secret or wrong environment |
| 403 | IP address is not allowed for this sunbitKey | register your egress IP with Sunbit |
| 404 | Location does not exist | location not on this account |
| 422 | Number of epays for sunbitKey passed the daily limit | unpublished per-key daily cap; contact Sunbit |
| 422 | Amount should be greater than 0 | |
| 422 | Customer reference number is invalid | the pre-qualification reference expired |
| 422 | Email address is not valid | bad `representativeEmail` |
| 500 | Internal Server Error | retry with backoff |

On validation: 403 `Forbidden purchaseId / transactionId`, 422 wrong purchase.

## Safety rules

- **Not idempotent.** No `Idempotency-Key`. Re-POSTing initialization with the same
  `transactionId` is not documented as safe — on timeout, call `GET /epay/api/v1/epay` with
  your `transactionId` and read the status before retrying.
- Amounts are USD, up to 2 decimals. There is no currency field.
- 403 means "bad credentials" *or* "not yours" *or* "IP blocked" — read the message.
