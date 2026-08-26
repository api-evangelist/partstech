---
name: partstech-quote-and-order
description: >-
  Build a PartsTech punchout cart, check live supplier availability and pricing, and place a wholesale
  parts order. THIS SKILL SPENDS REAL MONEY AND THE API HAS NO WAY TO TAKE IT BACK — read the
  irreversibility section before using it.
api: PartsTech External API
generated: '2026-08-26'
method: generated
source: openapi/partstech-api-openapi.yml
base_url: https://api.partstech.com
consequence: high
irreversible: true
operations:
  - getAcessToken
  - requestQuote
  - lineCardSearch
  - createCart
  - addPartToCart
  - updateCart
  - getCart
  - removeItemsFromCart
  - checkCartAvailability
  - submitCart
  - customCartCreate
  - checkItemsAvailability
  - submitItems
  - getPunchoutOrders
  - getOrders
  - getOrder
---

# Quote and order parts through PartsTech

## Read this first — there is no undo

`POST /punchout/cart/submit` (`submitCart`) and `POST /punchout/cart/custom/order` (`submitItems`)
place real purchase orders against the shop's real wholesale accounts with real suppliers.

**There is no cancel, void, return or refund operation anywhere in the 101-operation PartsTech
contract.** Once a submit returns success, the API gives you no way to reverse it. Cancellation, if it
is possible at all, happens on the phone with the supplier.

Therefore: never call `submitCart` or `submitItems` without an explicit, specific human confirmation of
the exact cart contents and total. "The user asked me to order brake pads" is not confirmation of a
$340 order across three suppliers.

There is also no idempotency key on this API. If a submit times out, **do not blind-retry it** — call
`POST /punchout/cart/info` (`getCart`) or `POST /punchout/orders` (`getPunchoutOrders`) to find out
whether the order landed. A retry may buy the parts twice.

## 1. Price before you commit

`POST /catalog/quote` (`requestQuote`) returns live wholesale pricing and availability from the shop's
suppliers for a set of parts. `POST /catalog/line-card-search` (`lineCardSearch`) searches a specific
store's line card. Both are read-only. Do all your comparison here.

## 2. Build the cart

- `POST /punchout/cart/create` (`createCart`) opens a punchout session and returns a `sessionId`.
- `POST /punchout/cart/add-part` (`addPartToCart`) adds a part.
- `POST /punchout/cart/update` (`updateCart`) changes quantities and cart-level fields.
- `POST /punchout/cart/info` (`getCart`) reads the current cart.
- `POST /punchout/cart/remove-parts` (`removeItemsFromCart`) removes parts. **This is the only reversal
  operation in the ordering flow.** It works while the session is open; the contract states no window,
  so treat the session as the boundary and do not assume you can come back later.

For items that are not in the PartsTech catalog, use the custom-cart pair instead:
`POST /punchout/cart/custom/create` (`customCartCreate`) then
`POST /punchout/cart/custom/availability` (`checkItemsAvailability`).

`409 LockedSession` means another operation on this session is still running. Wait and retry the same
call — do not open a second cart.

## 3. Rehearse

`POST /punchout/cart/availability` (`checkCartAvailability`) re-verifies availability and pricing across
every supplier in the cart. It is not a dry run of the write, but it is the only rehearsal this API
offers and it is the last cheap moment to catch a price change or a stock-out.

Handle the shaped errors it can return:

- `OrderAvailabilityError` — some orders cannot be purchased; `errorDetails` names the supplier, store
  and parts. Fix the cart, do not submit.
- `RequiredAdditionalParametersError` — the supplier needs extra fields filled in the cart; the response
  carries a `redirectUrl` a human must complete.
- `OrderNotPlacedError` — a credit-card transaction is required and must go through the cart UI; again a
  `redirectUrl`, again a human.
- `EmptyCartError` / `CartPartNotFoundError` — the cart drifted; re-read it with `getCart`.

## 4. Submit, then confirm

Call `submitCart` (or `submitItems` for a custom cart) exactly once. Then verify with
`POST /punchout/orders` (`getPunchoutOrders`) for the session, or `GET /orders` (`getOrders`) /
`GET /orders/{orderId}` (`getOrder`) for the shop's order history. Partners with partner-scoped
credentials read `GET /partner/orders` (`getOrdersByPartner`) instead.

## 5. Callbacks

If you supplied a `callbackUrl` / `callbackOrderUrl` when creating the session, PartsTech will POST the
quote or order payload to your endpoint. The `action` field discriminates: `SUBMIT_QUOTE` is a quote,
`PURCHASE` is a placed order, `TIRE_QUOTE` is a tire comparison quote. Note that these callbacks are
**not signed** and have no published retry policy — authenticate them some other way and make your
handler idempotent yourself.
