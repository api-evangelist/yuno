---
name: Accept a payment with Yuno
description: Create a customer, open a checkout session, and create a payment through Yuno's payment orchestration API, handling idempotency and declines correctly.
api: openapi/yuno-openapi-original.json
operations:
  - createCustomer
  - createCheckoutSession
  - createPayment
  - retrievePayment
---

# Accept a payment with Yuno

Yuno is a payment orchestration layer: one API routes to 1,000+ payment methods and
PSPs. Use the **sandbox** host while building.

## Environment & auth
- Sandbox base URL: `https://api-sandbox.y.uno` (Production: `https://api.y.uno`).
- Every request sends the header keys: `public-api-key`, `private-secret-key`
  (server-side only), and `account-code`. Sandbox and production use different keys.
- Content type is `application/json`; requests time out after 60 seconds.

## Idempotency (required for writes)
- Send a unique `X-Idempotency-Key` (UUID) on `createPayment`.
- Yuno stores the key + result for **24h**; identical retries return the original
  response. A concurrent duplicate returns `400 REQUEST_IN_PROCESS`. `400`/`500`
  responses do NOT cache the key, so a corrected request may reuse it.

## Steps
1. **createCustomer** — create (or reuse via `merchant_customer_id`) the customer
   you are charging.
2. **createCheckoutSession** — open a checkout session for that `customer_id`;
   this returns the `checkout_session` used to collect a payment method.
3. **createPayment** — create the payment referencing the `checkout_session`
   (workflow `SDK_CHECKOUT` requires `checkout_session_id` and `ott`; `DIRECT` and
   `REDIRECT` workflows are also supported). Send the `X-Idempotency-Key` header.
4. **retrievePayment** — poll the payment by `payment_id` for the terminal status.

## Interpreting the result
- Read the transaction `response_code`. `SUCCEEDED` = paid.
- On `DECLINED`, map `response_code` via `errors/yuno-decline-codes.yml`
  (e.g. `INSUFFICIENT_FUNDS`, `DO_NOT_HONOR`, `THREE_D_SECURE_REQUIRED`). Respect
  the normalized `merchant_advice_code` (MAC) for whether/when to retry — never
  blindly re-attempt a `DO_NOT_TRY_AGAIN`/`NO_RETRY_*` code.
- `THREE_D_SECURE_REQUIRED` means run the 3DS challenge before retrying.

## Testing
Use the Yuno Test Payment Gateway (sandbox only, no credentials). Test cards are
in `sandbox/yuno-sandbox.yml` — e.g. Visa `4507990000000002` → `SUCCEEDED`,
`4507990000000028` → `DECLINED_BY_BANK` (expiry `11/28`, CVV `123`).
