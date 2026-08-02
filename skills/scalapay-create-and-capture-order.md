---
name: Create and capture a Scalapay order
description: Create an instalment order, send the shopper to Scalapay Checkout, and capture the funds immediately once the payment is authorized.
api: openapi/scalapay-openapi-original.yml
generated: '2026-08-02'
method: generated
source: https://developers.scalapay.com/docs/standard-capture-flow
operations:
  - postOrders            # POST /v2/orders
  - getPaymentsByToken    # GET  /v2/payments/{token}
  - postPaymentsCapture   # POST /v2/payments/capture
---

# Create and capture a Scalapay order (immediate payment flow)

Scalapay is a buy-now-pay-later provider: the shopper pays in instalments, the merchant is settled
by Scalapay. This is the **immediate payment flow** — capture the full amount in one call after the
shopper authorizes.

> `operationId`s referenced here are supplied by `overlays/scalapay-overlay.yaml`; the published
> specification declares none. Every method + path below is verbatim from
> `openapi/scalapay-openapi-original.yml`.

## Before you start

- Base URL: `https://integration.api.scalapay.com` (sandbox) or `https://api.scalapay.com` (production).
- Auth: `Authorization: Bearer <secret api key>` on **every** request. Keys are environment-scoped and
  prefixed `sp_`. Using a sandbox key against production returns `401`.
- Content type: `application/json` on requests; responses are always JSON.
- Only EUR is supported, in 14 authorised European territories.

## Steps

### 1. Create the order — `postOrders` (`POST /v2/orders`)

Send `Idempotency-Key: <your uuid>` as a header. Required body fields: `totalAmount`, `consumer`,
`items`, `merchant`, `shipping`.

- `totalAmount` — `{amount: "187.95", currency: "EUR"}` (amount is a **string**).
- `consumer` — `givenNames`, `surname`, `email`, `phoneNumber` are all required.
- `items[]` — each needs `name`, `category`, `sku`, `quantity`, `price`.
- `merchant` — `redirectConfirmUrl` and `redirectCancelUrl`.
- `merchantReference` — set your own order id here. If you cannot yet, you can set it later with
  `postOrdersByToken` (`POST /v2/orders/{token}`).
- `product` — omit, or `pay-in-3` / `pay-in-4` / `later`.
- `orderExpiryMilliseconds` — how long the order stays valid; the ceiling is contractual.

Response: `{token, expires, checkoutUrl}`. **Persist `token` against your order** — it is the handle
for every later operation.

### 2. Redirect the shopper to `checkoutUrl`

The shopper authorizes on Scalapay Checkout and is returned to `redirectConfirmUrl` (or
`redirectCancelUrl`). Do not treat the redirect as proof of payment.

### 3. Confirm authorization

Either wait for the `authorized` webhook (see `asyncapi/scalapay-webhooks.yml`) or poll
`getPaymentsByToken` (`GET /v2/payments/{token}`). Webhooks are the recommended path; they are
retried with exponential backoff and are **not ordered**, so make your handler idempotent on
`orderToken` + `status`.

### 4. Capture — `postPaymentsCapture` (`POST /v2/payments/capture`)

Send the order `token` in the body, with an `Idempotency-Key` header. You may capture **up to** the
total specified at creation, and may supply an updated `merchantReference` at the same time.
On success the shopper's instalment plan is charged and funds move to the merchant account; the
`charged` webhook fires.

## Rules

- **Idempotency.** `postOrders`, `postOrdersByToken`, `postPaymentsCapture`, `postPaymentsByTokenDelay`,
  `postPaymentsByTokenRefund` and `postPaymentsByTokenVoid` all accept `Idempotency-Key`. On any retry,
  resend the **same** key — never a fresh one.
- **409 `conflicting_operation_in_progress`** means another operation is already running on this order.
  Back off and retry with the same key; do not create a second order.
- **422 `invalid_token`** means the order token does not resolve. Re-check the token from step 1.
- **Errors** are `{errorCode, errorId, message, httpStatusCode}`. Branch on `errorCode`, never on
  `message` — Scalapay states the message text may change. Log `errorId`; it is what support needs.
- **401** returns the bare string `"Unauthorized"`, not the error envelope. Handle that shape specially.
- If the order is authorized but never captured before `orderExpiryMilliseconds` elapses, it expires
  and the `expired` webhook fires.

## Test it first

Sandbox card `5200828282828210` (any CVV, any expiry) succeeds; `4000000000000341` fails. Test API key
`qhtfs87hjnc12kkos`. Test redirect URLs `https://portal.integration.scalapay.com/success-url` and
`/failure-url`. Full set in `sandbox/scalapay-sandbox.yml`.
