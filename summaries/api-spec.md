---
type: Interface Reference
title: REST API Specification
description: The billing-service exposes an HTTP REST API defined in the OpenAPI 3.0.3 specification (billing-openapi.yaml) and mounted using the Chi router in internal/api/plans.go.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/summaries/api-spec.md
tags:
- billing-service
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD//Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/billing-openapi.yaml
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/api/plans.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: /Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/billing-openapi.yaml:L1-L25 -->
<!-- anchor: internal/api/plans.go:L1-L22 -->
<!-- anchor: README.md:L1-L40 -->

# REST API Specification

The `billing-service` exposes an HTTP REST API defined in the OpenAPI 3.0.3 specification (`billing-openapi.yaml`) and mounted using the Chi router in `internal/api/plans.go`.

The HTTP routing layer is initialized via `api.Mount(r chi.Router, svc *plans.Service)`, delegating plan logic to `entities/plan-service`.

---

## Endpoints Overview

| Method | Path | Summary | Consumer / Purpose |
| :--- | :--- | :--- | :--- |
| `GET` | [[ap:kb-tw-digital-customer-portal/summaries/api-spec#get-v1-plans-policyId|customer-portal (/v1/plans/{policyId})]] | Get plan details, schedule, and instalment fees | `customer-portal` |
| `POST` | `/v1/plans` | Create a payment plan | Internal (or triggered via `policy.policy.issued`) |

---

## `GET /v1/plans/{policyId}`

Retrieves the active payment plan, instalment schedule, and calculated instalment fee for a specific policy.

- **Handler:** `internal/api/plans.go`
- **Primary Consumer:** `customer-portal`
- **Related ADR:** `decisions/adr-0001-billing-owns-plans`, `decisions/adr-0002-instalment-fee`

### Parameters

| Name | In | Type | Required | Description |
| :--- | :--- | :--- | :--- | :--- |
| `policyId` | `path` | `string` | Yes | Unique identifier of the policy |

### Response: `200 OK`

Returns an `application/json` payload representing the plan metadata and scheduled instalments stored in `entities/plan-model`.

#### Schema

```yaml
type: object
properties:
  plan_id:
    type: string
  frequency:
    type: string
    enum: [monthly, yearly]
  annual_premium_pence:
    type: integer
  fee_pence:
    type: integer
    description: Yearly instalment fee
  instalments:
    type: array
    items:
      type: object
      properties:
        number:
          type: integer
        due_date:
          type: string
        amount_pence:
          type: integer
        fee_pence:
          type: integer
        status:
          type: string
```

#### Field Descriptions

- `plan_id`: Unique identifier for the payment plan.
- `frequency`: Payment frequency (`monthly` or `yearly`).
- `annual_premium_pence`: Total annual policy premium in pence.
- `fee_pence`: Total yearly instalment fee in pence (calculated via rules detailed in [[concepts/instalment-fee-calculation]]).
- `instalments`: Ordered list of instalment items:
  - `number`: The sequence number of the instalment (e.g., 1 to 12 for monthly).
  - `due_date`: Date when the instalment collection is due.
  - `amount_pence`: Premium portion due for this instalment in pence.
  - `fee_pence`: Monthly fee portion assigned to this instalment in pence.
  - `status`: Collection status of the instalment (managed via [[concepts/arrears-and-collections]]).

---

## `POST /v1/plans`

Internal endpoint used to create a new payment plan for a policy.

- **Handler:** `internal/api/plans.go`
- **Trigger:** Internal invocations, normally triggered asynchronously via the `policy.policy.issued` Pub/Sub event (see `summaries/events-spec`).
- **Downstream Operations:** Requires policy data from `policy-admin` via [[entities/downstream-clients]].

### Response: `201 Created`

Returns HTTP status code `201` upon successful plan initialization and schedule generation.
