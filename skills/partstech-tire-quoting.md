---
name: partstech-tire-quoting
description: >-
  Find tire sizes for a vehicle on the PartsTech External API, quote tires from the shop's tire
  suppliers, and understand how tire comparison quotes reach the partner application.
api: PartsTech External API
generated: '2026-08-26'
method: generated
source: openapi/partstech-api-openapi.yml
base_url: https://api.partstech.com
operations:
  - getAcessToken
  - tireSizes
  - tiresSizesByVehicle
  - getTireSupplierPreferences
  - requestQuote
  - createQuote
  - updateQuote
  - getQuoteInformation
  - createStockOrderQuote
---

# Quote tires

## 1. Establish the size

`GET /catalog/tires/{vehicleId}` (`tiresSizesByVehicle`) returns the tire sizes that fit a resolved
PartsTech vehicle. Resolve the vehicle first (see the `partstech-vehicle-to-parts` skill) rather than
asking a human for a size they may read off a worn sidewall.

`GET /catalog/tires/sizes` (`tireSizes`) lists sizes without vehicle context — use it only when you are
matching a size the customer already stated.

## 2. Check which tire suppliers the shop can actually buy from

`GET /profile/shop/tire-suppliers` (`getTireSupplierPreferences`) returns the shop's configured tire
supplier accounts. This matters commercially: PartsTech's own plan pages state the free tier reaches
30+ in-network tire suppliers while the paid tiers reach the full 50+ network, so a shop on the free
plan will legitimately see fewer results and that is not a bug to escalate.

## 3. Quote

`POST /catalog/quote` (`requestQuote`) returns live pricing and availability.

The punchout quote family is separate and is what an SMS uses to hand a quote back and forth:

- `POST /punchout/quote/create` (`createQuote`)
- `POST /punchout/quote/update` (`updateQuote`)
- `POST /punchout/quote/info` (`getQuoteInformation`)
- `POST /punchout/quote/stock-order` (`createStockOrderQuote`)

## 4. The tire quote callback

When a shop's customer picks a tire from a PartsTech tire comparison quote, PartsTech POSTs the
selection to the partner's `callbackUrl` with `action: TIRE_QUOTE`. This only fires if a valid
`callbackUrl` was supplied; the contract explicitly warns that supplying a bad one "slows user's work
with cart with no effect".

The callback is unsigned and carries no retry guarantee. See `asyncapi/partstech-webhooks.yml`.

## Rules

- Quoting is read-only and safe. Ordering is not — tires are ordered through the same irreversible
  `submitCart` path as parts. See the `partstech-quote-and-order` skill before placing anything.
- Rate limits and the `{"error":{"code","message"}}` envelope apply exactly as everywhere else on this
  API.
