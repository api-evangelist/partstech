---
name: partstech-vehicle-to-parts
description: >-
  Resolve a vehicle from a VIN, license plate or year/make/model/engine drill-down on the PartsTech
  External API, then find the parts that fit it and pull live catalog detail. Read-only — this skill
  never spends money.
api: PartsTech External API
generated: '2026-08-26'
method: generated
source: openapi/partstech-api-openapi.yml
base_url: https://api.partstech.com
operations:
  - getAcessToken
  - decodeVin
  - decodePlate
  - getVehicleByVin
  - getYears
  - getMakes
  - getModels
  - getSubmodels
  - getEngines
  - getVehicle
  - getCategories
  - getSubcategories
  - getPartTypes
  - getPartTypeAttributes
  - searchParts
  - getPart
  - validateFitmentForPart
---

# Resolve a vehicle and find parts that fit it

## 1. Authenticate

`POST /oauth/access` (`getAcessToken`) exchanges partner + user credentials for a JWT. Send it on every
later call as `Authorization: Bearer <accessToken>`.

There is no refresh worth using: `POST /oauth/refresh` (`refreshAccessToken`) is marked deprecated and
the contract tells you to generate a new token instead when the old one expires.

## 2. Resolve the vehicle

Pick the cheapest path that matches what you were given.

- **You have a VIN** — `GET /catalog/vin/{vin}` (`decodeVin`) or `GET /taxonomy/vehicles/vin/{vin}`
  (`getVehicleByVin`). Both take a 17-character ISO 3779 VIN and return a PartsTech `vehicleId`.
- **You have a plate** — `GET /catalog/plate/{state}/{plate}` (`decodePlate`).
- **You have neither** — drill down through VCdb: `getYears` → `getMakes` → `getModels` →
  `getSubmodels` → `getEngines`, then `GET /taxonomy/vehicles/{vehicleID}` (`getVehicle`).

Do not ask a human to disambiguate a vehicle you can resolve from a VIN. Do not guess a `vehicleId`:
every id space in this API is opaque and unprefixed, so a wrong-space integer will be accepted and will
silently return the wrong vehicle's parts.

## 3. Pick the part type

Parts are addressed through the Auto Care PCdb taxonomy, not free text.

`getCategories` → `getSubcategories` → `getPartTypes` narrows to a `partTypeId`.
`getPartTypeAttributes` tells you which attributes that part type accepts, which is how you constrain a
search (position, size, grade) instead of over-fetching.

## 4. Search and inspect

`POST /catalog/search` (`searchParts`) takes the vehicle and part-type context and returns catalog
parts. **This operation is flagged `deprecated: true` in the contract with no named successor** — it
still works, it is still the primary search, but confirm the replacement with
`PartsTech-Partner-API@oeconnection.com` before you build a long-lived integration on it.

`GET /catalog/parts/{partId}` (`getPart`) — or `POST /catalog/parts/{partId}`
(`getPartWithRequestBody`) when you need to pass vehicle context in a body — returns full part detail:
brand, taxonomy, images, attributes, variations.

`POST /catalog/parts/{partId}/validate-fitment` (`validateFitmentForPart`) confirms a part fits a
vehicle. Run it before you put anything in a cart. A 204 is a valid answer here, not an error.

## 5. Rules that apply to every call in this skill

- **Errors are not RFC 9457.** Every failure is `{"error":{"code":"...","message":"..."}}`. Branch on
  `error.code`, never on the message string. `403 IncorrectMode` means you used a live user against the
  beta environment or a test user against production. `403 DisabledApiUsage` means the shop has not
  approved API access — that is a human task, not a retry.
- **Watch `x-rate-limit-remaining`.** Limits apply per day, per second, concurrently and over a sliding
  15 minutes, and PartsTech does not publish the numbers. On `429 TooManyRequests` there is no
  `Retry-After`; back off until the `x-rate-limit-reset` epoch-seconds timestamp.
- **`432` is not a real HTTP status.** It is PartsTech's integration-error code for a supplier-side
  failure. Do not let a generic HTTP client treat it as unknown-and-retryable.
- **Log `x-trace-id`.** It is undocumented but present on every response and it is what support needs.
- Pagination, where it exists, is `page` / `perPage` with an RFC 8288 `Link` header carrying
  `first`/`previous`/`next`/`last`.
