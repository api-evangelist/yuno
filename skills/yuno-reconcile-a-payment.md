---
name: Retrieve and reconcile a payment
description: Retrieve a Yuno payment by id, interpret its status/response_code, and reconcile against webhook events.
api: openapi/yuno-openapi-original.json
operations:
  - retrievePayment
---

# Retrieve and reconcile a payment

## Auth & environment
- Send `public-api-key`, `private-secret-key`, `account-code` headers.
- Sandbox `https://api-sandbox.y.uno`, Production `https://api.y.uno`.

## Steps
1. **retrievePayment** — fetch the payment by `payment_id`.
2. Read the payment/transaction **status** and **`response_code`**:
   - Terminal success: `SUCCEEDED`.
   - In-flight: `CREATED` (may need user action), `PENDING` (awaiting confirmation
     or fraud review) — poll again or wait for the webhook.
   - Failure: `DECLINED` / `REJECTED` / `ERROR` / `EXPIRED`.
3. For failures, resolve the `response_code` and `merchant_advice_code` against
   `errors/yuno-decline-codes.yml` and `errors/yuno-problem-types.yml`.

## Reconcile with webhooks
Do not rely on polling alone. Subscribe to the webhook events in
`asyncapi/yuno-webhooks.yml` — `payment.purchase`, `payment.refund`,
`payment.chargeback` — and treat the webhook as the source of truth for terminal
state. Webhook delivery can be OAuth2-authenticated; chargebacks can be routed to
a dedicated URL. Handle retries and de-duplicate by event id.
