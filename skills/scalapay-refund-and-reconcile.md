---
name: Refund a Scalapay order and reconcile the payout
description: Refund a fulfilled instalment payment, then tie orders, refunds and Scalapay fees back to the bank transfer that settled them.
api: openapi/scalapay-openapi-original.yml
generated: '2026-08-02'
method: generated
source: https://developers.scalapay.com/reference/get_v1-reporting-payouts
operations:
  - postPaymentsByTokenRefund        # POST /v2/payments/{token}/refund
  - getReportingRefunds              # GET  /v1/reporting/refunds
  - getReportingOrders               # GET  /v1/reporting/orders
  - getReportingPayouts              # GET  /v1/reporting/payouts
  - getReportingPayoutsByTokenOrders # GET  /v1/reporting/payouts/{token}/orders
  - getReportingPayoutsByTokenRefunds# GET  /v1/reporting/payouts/{token}/refunds
  - getPaymentsReferences            # GET  /v2/payments/references
---

# Refund and reconcile

Two jobs that share one data spine: reversing money on a captured order, and matching Scalapay's bank
transfers to the orders and refunds inside them.

> `operationId`s come from `overlays/scalapay-overlay.yaml`; the published spec declares none.
> Reporting endpoints authenticate with the **merchant** Scalapay API key bearer token.

## Refund

### 1. Find the order token

If you only hold your own reference, use `getPaymentsReferences`
(`GET /v2/payments/references`) with any of `orderToken`, `merchantOrderReference`,
`merchantProcessorReference`. At least one is required; multiple are ANDed. It returns at most
**100** orders, newest first.

### 2. Refund — `postPaymentsByTokenRefund` (`POST /v2/payments/{token}/refund`)

Only works on a **fulfilled** (captured) payment — funds reverse from the merchant account back to the
customer. Send `Idempotency-Key`; on any retry resend the same key, or you risk a second refund.
The `refunded` webhook fires on success.

> If the payment was authorized but not yet captured, do **not** refund — `postPaymentsByTokenVoid`
> (`POST /v2/payments/{token}/void`) is the correct call.

## Reconcile

### 3. List payouts — `getReportingPayouts` (`GET /v1/reporting/payouts`)

Query with `startDate`, `endDate`, `page`, `size`. Each payout carries `merchantPayoutToken`,
`transactionDate`, `status`, and the money breakdown: `grossAmount`, `netAmount`, `totalFeeAmount`,
`scalapayFeeAmount`, `scalapayFeeTaxAmount`, `otherFeeAmount`, `otherFeeTaxAmount`.

### 4. Explode each payout

- `getReportingPayoutsByTokenOrders` (`GET /v1/reporting/payouts/{token}/orders`) — the orders settled
  by that payout.
- `getReportingPayoutsByTokenRefunds` (`GET /v1/reporting/payouts/{token}/refunds`) — the refunds
  deducted by it.

Both take `page` and `size`.

### 5. Cross-check the period

`getReportingOrders` (`GET /v1/reporting/orders`) and `getReportingRefunds`
(`GET /v1/reporting/refunds`) list everything in a date window. Order rows carry `orderStatus`
(`created` / `authorized` / `charged` / `refunded` / `expired`), `captureStatus`
(`captured` / `delayed`), `captureAmount`, `merchantReference`, `transferId` and a nested
`payoutDetails`, so an order row can be walked straight to its payout.

> Order tokens are **masked** in reporting — `orderTokenLast4` shows `******AYPA`. Join on
> `merchantReference`, not on the token.

## Reconciliation loop for an agent

1. Pull payouts for the period.
2. For each payout, sum settled orders minus refunds, minus `totalFeeAmount` — that should equal
   `netAmount`.
3. Anything in `getReportingOrders` with `orderStatus: charged` and no `payoutDetails` is captured but
   not yet settled — expected, not an error.
4. `getReportingDisputes` (`GET /v1/reporting/disputes`) covers disputed orders by `startDate`,
   `endDate` and `disputeStatus`. It is in the spec but has no page in the published reference, so
   treat its response shape as unverified.

## Errors

`400 pre_condition_failed` is the reporting workhorse — `startDate` after `endDate`, or an invalid
payout token. `500 internal_server` on disputes: retry with backoff and quote `errorId`.
See `errors/scalapay-error-codes.yml`.
