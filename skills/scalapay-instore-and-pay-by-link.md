---
name: Take a Scalapay in-store or pay-by-link order
description: Charge a customer at a physical point of sale, or send them a pay-by-link instalment order over SMS or email, using a device-scoped API key.
api: openapi/scalapay-openapi-original.yml
generated: '2026-08-02'
method: generated
source: https://developers.scalapay.com/docs/integration-in-store
operations:
  - postInstoreOrders                      # POST /v1/instore/orders
  - getInstoreOrdersByToken                # GET  /v1/instore/orders/{token}
  - getInstoreOrdersReferences             # GET  /v1/instore/orders/references
  - postInstoreOrdersByTokenRefund         # POST /v1/instore/orders/{token}/refund
  - postInstorePaybylinkOrders             # POST /v2/instore/paybylink/orders
  - postInstorePaybylinkOrdersByTokenVoid  # POST /v2/instore/paybylink/orders/{token}/void
---

# In-store and pay-by-link orders

The in-store family is a separate resource space from online `/v2/orders`, with **its own credential**.

> **Use the device bearer token, not the merchant token.** The OpenAPI declares a second security
> scheme, `InstoreApiKeyAuth`, for these endpoints. Sending the merchant key here returns `401`
> (bare string `"Unauthorized"`, not the error envelope). Sandbox device key: `testdeviceapikey`.

> `operationId`s come from `overlays/scalapay-overlay.yaml`; the published spec declares none.

## A. In-store order at the till

### 1. `postInstoreOrders` (`POST /v1/instore/orders`)

Creates the in-store order **and charges the customer** in one call. There is no separate capture step
and no `Idempotency-Key` parameter on this operation — so treat it as **not safely retryable**: on a
timeout, do **not** blind-retry. Reconcile first with step 2 or 3.

### 2. `getInstoreOrdersByToken` (`GET /v1/instore/orders/{token}`)

Retrieve the status of an in-store order.

### 3. `getInstoreOrdersReferences` (`GET /v1/instore/orders/references`)

Recover an order you have lost the token for: query by `orderToken`, `merchantOrderReference` or
`merchantProcessorReference`. At least one required, multiple ANDed, max **100** results, newest
first. This is the correct recovery move after an ambiguous timeout.

### 4. `postInstoreOrdersByTokenRefund` (`POST /v1/instore/orders/{token}/refund`)

Processes the refund and returns funds to the customer.

## B. Offline pay-by-link

### 1. `postInstorePaybylinkOrders` (`POST /v2/instore/paybylink/orders`)

Creates an offline pay-by-link order and notifies the customer. Notification is configured through the
`extensions` object:

```
extensions.type.link.notification.channels        # required
extensions.type.link.notification.phoneCountryCode # e.g. "+39" — required if SMS is a channel
extensions.type.link.notification.phoneNumber
```

Again: **device** bearer token, not the merchant token.

### 2. `postInstorePaybylinkOrdersByTokenVoid` (`POST /v2/instore/paybylink/orders/{token}/void`)

Voids the order — for example when the customer cancels. Any authorizations or held funds for the
pay-by-link order are released.

## Rules

- Device key for every operation in this skill; merchant key for reporting and online `/v2` flows.
- Travel merchants pass `extensions.industry.travel.startDate` / `.endDate` on the order.
- Only EUR, and only in the 14 authorised territories (see `lifecycle/scalapay-lifecycle.yml`).
- These orders surface in `/v1/reporting/*` alongside online orders, keyed on `merchantReference`
  (see `scalapay-refund-and-reconcile.md`). Reporting uses the **merchant** key.
- Errors: `errors/scalapay-error-codes.yml`. Sandbox values: `sandbox/scalapay-sandbox.yml`.
