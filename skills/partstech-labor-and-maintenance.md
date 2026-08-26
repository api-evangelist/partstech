---
name: partstech-labor-and-maintenance
description: >-
  Pull MOTOR- and Mitchell 1-powered labor times, maintenance schedules, fluid specifications and
  vehicle specifications from the PartsTech External API, and honor the OEM compliance requirement that
  rides on those responses.
api: PartsTech External API
generated: '2026-08-26'
method: generated
source: openapi/partstech-api-openapi.yml
base_url: https://api.partstech.com
operations:
  - getAcessToken
  - getLabor
  - getLaborDetails
  - getJobs
  - maintenanceSchedulesIndicators
  - maintenanceSchedulesDetailsByIndicator
  - maintenanceSchedulesIntervals
  - maintenanceSchedulesDetailsByInterval
  - maintenanceSchedulesDetails
  - fluidsSummary
  - fluidsDetails
  - contentSilos
  - specificationsSummary
  - specificationsDetails
  - GetMitchell1VehicleId
  - GetGroups
  - GetSubGroups
  - GetProceduresSummary
  - SearchProceduresSummary
  - GetProcedure
---

# Labor, maintenance and specifications content

PartsTech resells two licensed content sets through this API. They are separate surfaces with separate
vehicle identifiers — do not mix them.

## MOTOR surface

Resolve the PartsTech vehicle first, then:

- **Labor** — `POST /taxonomy/labor` (`getLabor`) lists labor operations; `POST /taxonomy/labor/details`
  (`getLaborDetails`) expands one by operation id. `GET /taxonomy/jobs` (`getJobs`) lists the canned job
  templates that map a repair to its part types.
- **Maintenance schedules** — indicator-first (`maintenanceSchedulesIndicators` →
  `maintenanceSchedulesDetailsByIndicator`) or interval-first (`maintenanceSchedulesIntervals` →
  `maintenanceSchedulesDetailsByInterval`), with `maintenanceSchedulesDetails` fetching by application
  id.
- **Fluids** — `POST /fluids/summary` (`fluidsSummary`) then `POST /fluids/details` (`fluidsDetails`).
- **Specifications** — `POST /specifications/content-silos` (`contentSilos`) enumerates the silos, then
  `specificationsSummary` and `specificationsDetails`.

These endpoints are keyed on a MOTOR `applicationId`, which is a different id space from the PartsTech
`vehicleId`. Carry the application id through the summary → details pair; do not construct one.

### The compliance Link header is not optional decoration

Every MOTOR-powered response returns an RFC 8288 `Link` header:

```
Link: <https://www.motor.com/oem-compliance-requirements/>; rel="compliance"
```

That header is a licensing signal attached to the data. If you cache, redisplay or re-syndicate MOTOR
content, read what it points at and comply with it. Stripping it because your HTTP client only keeps
the body is the wrong default.

## Mitchell 1 surface

Mitchell 1 has its **own** vehicle id. Get it with `POST /mitchell1/vehicles`
(`GetMitchell1VehicleId`), then walk `POST /mitchell1/labor/groups` (`GetGroups`) →
`/mitchell1/labor/subgroups` (`GetSubGroups`) → `/mitchell1/labor/procedures`
(`GetProceduresSummary`) → `/mitchell1/labor/details` (`GetProcedure`).
`POST /mitchell1/labor/search` (`SearchProceduresSummary`) searches procedures directly.

Passing a PartsTech `vehicleId` to a `/mitchell1/*` endpoint will not error usefully — it is a different
id space. Always start with `GetMitchell1VehicleId`.

## Rules

- Everything in this skill is read-only.
- All of these endpoints declare `400` and `404` responses; the envelope is
  `{"error":{"code","message"}}`, not RFC 9457.
- Access to MOTOR and Mitchell 1 content depends on the partner's entitlement. A `403 NotAvailableMethod`
  or `403 HaveNotPermission` here means the entitlement is missing — a commercial conversation with
  `PartsTech-Partner-API@oeconnection.com`, not a retry.
