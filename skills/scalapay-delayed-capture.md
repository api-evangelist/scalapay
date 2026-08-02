---
name: Run a Scalapay delayed-capture order
description: Authorize an instalment order now and capture it later when stock ships, voiding the authorization if the order cannot be fulfilled.
api: openapi/scalapay-openapi-original.yml
generated: '2026-08-02'
method: generated
source: https://developers.scalapay.com/docs/delayed-capture-flow
operations:
  - postOrders                # POST /v2/orders
  - postPaymentsByTokenDelay  # POST /v2/payments/{token}/delay
  - postPaymentsCapture       # POST /v2/payments/capture
  - postPaymentsByTokenVoid   # POST /v2/payments/{token}/void
  - getPaymentsByToken        # GET  /v2/payments/{token}
---

# Delayed-capture order

Use this when you must not take money until the goods actually ship. Scalapay creates the shopper's
instalment schedule at delay time, but **no funds settle until you capture**.

> `operationId`s come from `overlays/scalapay-overlay.yaml`; the published spec declares none.

## Steps

### 1. Create the order — `postOrders` (`POST /v2/orders`)

Same body as the immediate flow (see `scalapay-create-and-capture-order.md`). Send an
`Idempotency-Key` header. Keep the returned `token`, and redirect the shopper to `checkoutUrl`.

### 2. Wait for authorization

The `authorized` webhook fires when the shopper authorizes on the Scalapay page. You can also read
`getPaymentsByToken` (`GET /v2/payments/{token}`).

### 3. Request the delay — `postPaymentsByTokenDelay` (`POST /v2/payments/{token}/delay`)

Send `Idempotency-Key`. This creates the customer's payment schedule **without settling funds**.
The order shows `captureStatus: delayed` in reporting.

> Outstanding authorizations are automatically voided once the authorization expiry time is reached.
> Capture before that, or you lose the authorization.

### 4a. Ship it — capture — `postPaymentsCapture` (`POST /v2/payments/capture`)

Body carries the order `token`; header carries `Idempotency-Key`. You may capture **up to** the
original total — a partial capture is allowed, so capture the amount you actually shipped. You may
also pass an updated `merchantReference`. The `charged` webhook follows.

### 4b. Cannot fulfil — void — `postPaymentsByTokenVoid` (`POST /v2/payments/{token}/void`)

Voids any authorizations and releases held funds. The customer is told the order was cancelled and
could not be fulfilled. Send `Idempotency-Key`.

## Decision rule for an agent

| Situation | Call |
|---|---|
| Stock confirmed, shipping now | `postPaymentsCapture` for the shipped amount |
| Partial shipment | `postPaymentsCapture` for the shipped subtotal only |
| Cannot fulfil at all | `postPaymentsByTokenVoid` |
| Already captured, goods returned | `postPaymentsByTokenRefund` (see `scalapay-refund-and-reconcile.md`) |

Never call capture and void for the same token. Never issue a second `postOrders` to "retry" — retry
the original call with the **same** `Idempotency-Key`.

## Errors

- `409 conflicting_operation_in_progress` — another operation is in flight on this order; back off,
  retry with the same key.
- `422 invalid_token` — the token does not resolve.
- `400 api_validationerror` — read `message.errors[]` for the offending `field` / `messages`.
- `401` returns the bare string `"Unauthorized"`, not the standard envelope.

Full catalogue: `errors/scalapay-error-codes.yml`. Conventions: `conventions/scalapay-conventions.yml`.
Sandbox values: `sandbox/scalapay-sandbox.yml`.
