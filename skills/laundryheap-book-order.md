---
name: laundryheap-book-order
description: >-
  Book a Laundryheap laundry or dry-cleaning collection and delivery — resolve a
  serviceable address, pick services, choose pickup and dropoff windows, and
  create the order.
api: Laundryheap GraphQL API
endpoint: https://www.laundryheap.com/graphql
operations:
  - validateAddressWithMappedCity
  - services
  - getPaymentInfo
  - timeslotsV2
  - createAddress
  - createGuestUser
  - createOrder
  - applyPromoCode
provenance:
  method: generated
  grounded_in: graphql/laundryheap-graphql.yml
  source: https://app.laundryheap.com/assets/index-BQ9DuQOF.js
  generated: '2026-08-23'
  caveat: >-
    Every field and argument named here was extracted from Laundryheap's own
    production client bundle and cross-checked against the live endpoint.
    Laundryheap publishes no API documentation, no schema (introspection is
    disabled) and no process for obtaining credentials, so return shapes and
    selection sets are deliberately not specified below.
---

# Book a Laundryheap order

## Before you start

Laundryheap has no developer portal. You need either an authenticated first-party session or an
OAuth 2.0 client with the `orders.create` scope from
`https://www.laundryheap.com/.well-known/openid-configuration`. There is no published process for
obtaining one; dynamic client registration is open at `/oauth/registration` but its terms are not
stated. Business contact: collaborate@laundryheap.com.

All calls are `POST https://www.laundryheap.com/graphql` with `Content-Type: application/json`.

## Step 1 — check the address is serviceable

Geocode the customer's address yourself, then validate it:

```graphql
mutation ($latitude: Float!, $longitude: Float!, $serviceCountry: ServiceCountryEnum) {
  validateAddressWithMappedCity(latitude: $latitude, longitude: $longitude, serviceCountry: $serviceCountry) { ... }
}
```

`serviceCountry` is a `ServiceCountryEnum` and it partitions almost everything in this API — carry it
through every subsequent call. If you only need to know whether an address is covered at all, the
unauthenticated REST endpoint answers without credentials:

```
GET https://www.laundryheap.com/api/v1/services?address=London
```

A serviceable address returns `200` with the service catalogue. An unserviceable one returns `404`
with `{"code":"not_found","message":"Address not found"}`. Note that a `404` carrying
`{"status":404,"error":"Not Found"}` instead means the *endpoint* does not exist — a different
failure entirely.

## Step 2 — list the services available there

```graphql
query ($serviceCountry: ServiceCountryEnum, $latitude: Float, $longitude: Float, $onlyExpress: Boolean) {
  services(serviceCountry: $serviceCountry, latitude: $latitude, longitude: $longitude, onlyExpress: $onlyExpress) { ... }
}
```

Services are identified by `short_code`. The seven observed codes are `mixed_wash`, `separate_wash`,
`wash_and_iron`, `dry_cleaning`, `ironing`, `duvets_bulky_items` and `repairs`. Each carries a
`minimum_service_minutes` — 1440 (24h) for the standard services, 4320 (72h) for duvets, bulky items
and repairs. Do not offer a dropoff window inside a service's own minimum.

## Step 3 — save the address if it is new

```graphql
mutation ($countryCode: String!, $latitude: Float!, $longitude: Float!, $city: String, $country: String, $isDefault: Boolean) {
  createAddress(countryCode: $countryCode, latitude: $latitude, longitude: $longitude, city: $city, country: $country, isDefault: $isDefault) { ... }
}
```

Latitude and longitude are required — this API does not accept an unresolved address string. For a
customer without an account, `createGuestUser(serviceCountry!, email!, firstName, lastName, phone)`
creates one without a password.

## Step 4 — find pickup and dropoff windows

```graphql
query ($type: AvailabilityTypeEnumType!, $pickupAfterTime: Time, $pickupBeforeTime: Time,
       $dropoffAfterTime: Time, $dropoffBeforeTime: Time, $serviceIds: [Int!]) {
  timeslotsV2(type: $type, pickupAfterTime: $pickupAfterTime, pickupBeforeTime: $pickupBeforeTime,
              dropoffAfterTime: $dropoffAfterTime, dropoffBeforeTime: $dropoffBeforeTime,
              serviceIds: $serviceIds) { ... }
}
```

Use `timeslotsV2`, not any unsuffixed predecessor. Laundryheap versions by adding a new field beside
the old one and does not mark the old one deprecated, so prefer the highest-numbered variant of any
field.

## Step 5 — price it before committing

```graphql
query ($serviceCountry: ServiceCountryEnum!, $addressID: Int, $lat: Float, $lng: Float,
       $addressType: String, $udprn: Int) {
  getPaymentInfo(serviceCountry: $serviceCountry, addressID: $addressID, lat: $lat, lng: $lng,
                 addressType: $addressType, udprn: $udprn) { ... }
}
```

This is the closest thing to a dry run — there is no rehearsal mode on `createOrder` itself. Tell the
customer the price is an estimate: Laundryheap itemises the order after collection and the final
invoice can differ.

## Step 6 — create the order

```graphql
mutation ($pickupAfterDateTime: String!, $pickupBeforeDateTime: String!,
          $dropOffAfterDateTime: String!, $dropOffBeforeDateTime: String!) {
  createOrder(pickupAfterDateTime: $pickupAfterDateTime, pickupBeforeDateTime: $pickupBeforeDateTime,
              dropOffAfterDateTime: $dropOffAfterDateTime, dropOffBeforeDateTime: $dropOffBeforeDateTime) { ... }
}
```

All four window bounds are required. Keep the returned order `uuid` — it is an opaque string and it
is the handle for every later operation.

**There is no idempotency mechanism.** No `Idempotency-Key` header and no idempotency argument exists
on this mutation. If the call times out, do NOT blindly retry: query `ordersV2` first to check
whether the order was in fact created, or you will book the customer twice.

## Step 7 — apply a promo code (optional)

```graphql
mutation ($uuid: String!, $promoCode: String!) { applyPromoCode(uuid: $uuid, promoCode: $promoCode) { ... } }
```

Validate first with `validPromotion(promoCode!, serviceCountry, simplifiedBooking)`. Reverse with
`removePromoCode(uuid!)`.

## Errors

Transport is `200` even on failure — read `errors[]`.

| Signal | Meaning | Do |
|---|---|---|
| `extensions.short_code: "authentication"` | Not authenticated | Re-authenticate; do not retry as-is |
| `extensions.code: "undefinedField"` | Field is not in the schema | Read the did-you-mean hint; the schema changed |
| `extensions.code: "selectionMismatch"` | Composite type needs a selection set | Fix the query |

There are no rate-limit headers on any response, so there is no signal to pace against. Be
conservative and back off on your own schedule.

## Taking it back

`cancelOrder(uuid, reasons)` cancels free of charge up to **2 hours** before the scheduled collection
or delivery. Inside that window the cancellation still succeeds but a fee is added to the invoice.
Tell the customer the deadline at booking time. See `laundryheap-manage-order`.
