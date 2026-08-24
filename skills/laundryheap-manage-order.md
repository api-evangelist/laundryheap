---
name: laundryheap-manage-order
description: >-
  Inspect, reschedule and cancel an existing Laundryheap order, and manage
  recurring orders — including the 2-hour cancellation window an agent must
  respect before it acts.
api: Laundryheap GraphQL API
endpoint: https://www.laundryheap.com/graphql
operations:
  - ordersV2
  - order
  - modifyOrder
  - updateOrderServices
  - cancelOrder
  - getPaymentPolicy
  - sendInvoiceMail
  - repeatOrders
  - repeatOrder
  - repeatOrderTimeslots
  - repeatOrderUpdate
  - repeatOrderUpdateSchedule
  - repeatOrderDelete
provenance:
  method: generated
  grounded_in: graphql/laundryheap-graphql.yml
  source: https://app.laundryheap.com/assets/index-BQ9DuQOF.js
  generated: '2026-08-23'
  caveat: >-
    Field and argument names are verbatim from Laundryheap's published production
    client bundle. Laundryheap publishes no API documentation and no schema, so
    return shapes are not specified. The cancellation window is quoted from the
    Laundryheap Help Centre, which is a published source.
---

# Manage a Laundryheap order

## The clock, first

Before doing anything else, work out how long the customer has:

> "We offer the option to make delivery changes or cancel your order up to 2 hours before the
> delivery/collection at no extra cost. However, please keep in mind if the request to cancel is
> submitted within 2 hours of the collection, a fee will be added to your invoice."
> — [Laundryheap Help Centre](https://help.laundryheap.com/en/articles/6265252-what-if-i-need-to-change-my-delivery-preferences-or-cancel-my-order)

Compare `now` against the order's pickup window. If you are inside 2 hours, **say so and get explicit
confirmation before cancelling** — the action still succeeds, but it costs the customer money.
Laundryheap also reserves the right to charge for rescheduling requested inside the 2-hour window or
outside office hours (09:00–18:00 weekdays).

## List orders

```graphql
query ($page: Int, $perPage: Int, $sortBy: OrderSortEnum, $sortDirection: SortDirectionEnum, $state: OrderStateEnumType!) {
  ordersV2(page: $page, perPage: $perPage, sortBy: $sortBy, sortDirection: $sortDirection, state: $state) { ... }
}
```

`state` is required — you cannot list "all orders" without naming a state. Pagination is page-number
based, not cursor based. Use `ordersV2`, not the older `orders` field.

## Fetch one order

```graphql
query ($uuid: String!) { order(uuid: $uuid) { ... } }
```

Orders are keyed by opaque `uuid` strings, not integers. `getPaymentPolicy(uuid!)` returns the
payment terms attached to that specific order — read it before you change anything that costs money.
`sendInvoiceMail(uuid!)` re-sends the invoice.

## Reschedule

```graphql
mutation ($uuid: String, $pickupAfterDateTime: String, $pickupBeforeDateTime: String,
          $dropOffAfterDateTime: String, $dropOffBeforeDateTime: String) {
  modifyOrder(uuid: $uuid, pickupAfterDateTime: $pickupAfterDateTime, pickupBeforeDateTime: $pickupBeforeDateTime,
              dropOffAfterDateTime: $dropOffAfterDateTime, dropOffBeforeDateTime: $dropOffBeforeDateTime) { ... }
}
```

Prefer rescheduling to cancel-and-rebook: it keeps the order id, the promo code and the payment
policy, and a rebook would re-enter the pricing flow from scratch.

## Change the services on an order

```graphql
mutation ($uuid: String!, $services: [OrderServiceInput]!, $selfItemizationItems: [IdWithQtyInput!]) {
  updateOrderServices(uuid: $uuid, services: $services, selfItemizationItems: $selfItemizationItems) { ... }
}
```

`selfItemizationItems` is how a customer declares the items themselves rather than letting the
facility itemise on receipt. Changing services changes the price.

## Cancel

```graphql
mutation ($uuid: String, $reasons: [OrderCancelReasonInput]) {
  cancelOrder(uuid: $uuid, reasons: $reasons) { ... }
}
```

Supply structured `reasons`. **Cancellation is not itself reversible** — there is no un-cancel
mutation. Confirm the 2-hour position with the customer before firing it.

## Recurring orders

Recurring orders are a separate entity with their own uuid space.

| Action | Field |
|---|---|
| List | `repeatOrders` |
| Fetch one | `repeatOrder(uuid!)` |
| Available windows | `repeatOrderTimeslots(pickupAfterTime, pickupBeforeTime, dropoffAfterTime, dropoffBeforeTime, serviceIds)` |
| Change details | `repeatOrderUpdate(addressId, dropoffAfterDateTime, dropoffBeforeDateTime, dropoffMethod, pickupAfterDateTime, ...)` |
| Change schedule only | `repeatOrderUpdateSchedule(input: UpdateScheduleInput!)` |
| Stop | `repeatOrderDelete(uuid!)` |

`repeatOrderDelete` stops future occurrences. Laundryheap does **not** document whether an
already-scheduled occurrence inside the 2-hour window is also cancelled — check `ordersV2` afterwards
and cancel any live occurrence explicitly rather than assuming.

## What you cannot undo

Do not treat these as reversible; none has a reversal field and no refund window is published:

- `buyCredits` — `claimCredits` is a redemption, not a refund.
- `purchaseBundle`
- `createSubscription` / `updateSubscription` — no cancel field exists on this surface.
- `postServiceReview` / `postDriverPickupReview` — no edit and no delete.

## Complaints are not an API action

Quality complaints go through the mobile app only, within 48 hours of delivery, and are reviewed
case by case. There is no mutation for this and no developer support channel — the Help Centre
explicitly declines phone and email support. `resolutionCenterContactUs(input: ContactUsInput!)`
exists in the client but is a contact form, not a claim.

## Errors and retries

Transport is always `200`; read `errors[]`. `extensions.short_code: "authentication"` means the
session or token is not valid. There is **no idempotency mechanism and no rate-limit header** on any
Laundryheap surface — before retrying any write, re-read `order(uuid)` to see whether the first
attempt landed.
