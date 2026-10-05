---
type: Component
title: Store
description: Store (internal/plans/store.go) is the persistence layer responsible for reading and writing payment plans and instalment schedules to the relational database.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/store.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/store.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/model/plan.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/cmd/billing/main.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: internal/plans/store.go:L1-L17 -->
<!-- anchor: internal/plans/model/plan.go:L1-L33 -->
<!-- anchor: cmd/billing/main.go:L1-L30 -->

# Store

`Store` (`internal/plans/store.go`) is the persistence layer responsible for reading and writing payment plans and instalment schedules to the relational database.

## Responsibilities

`Store` manages database operations across the `payment_plans` and `instalments` tables:

- **Save Plan and Schedule (`Save`):** Persists a newly created [[entities/plan|model.Plan]] entity alongside its full slice of schedule entries (`[]model.Instalment`) to `payment_plans` and `instalments`.
- **Query Plan by Policy (`ByPolicy`):** Retrieves the active [[entities/plan|model.Plan]] associated with a given `policyID` string.
- **Close Plan (`Close`):** Updates a plan record (`planID`) to mark it as closed, applying the calculated pro-rata cancellation refund amount (`refundPence int64`).
- **Mark Instalment as Collected (`MarkCollected`):** Updates the status of a specific instalment identified by `planID` and instalment `number` to reflect successful collection.

### Database Tables & Model Mappings

`Store` operates on two primary tables defined in `internal/plans/model/plan.go`:

#### `payment_plans` Table
Maps to `model.Plan`:
| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique identifier of the payment plan |
| `policy_id` | `string` | Identifier of the associated policy from `policy-admin` |
| `customer_id` | `string` | Identifier of the customer |
| `frequency` | `model.Frequency` | Payment frequency (`monthly` or `yearly`) |
| `annual_premium_pence` | `int64` | Policy annual premium in pence |
| `instalments` | `int` | Number of scheduled instalments (e.g. 1 or 12) |
| `fee_pence` | `int64` | Total instalment fee charged in pence |
| `start_date` | `time.Time` | Policy and plan start date |
| `status` | `string` | Plan status: `active`, `in_arrears`, or `closed` |

#### `instalments` Table
Maps to `model.Instalment`:
| Field | Type | Description |
|---|---|---|
| `plan_id` | `string` | Foreign key referencing `payment_plans.id` |
| `number` | `int` | Sequential instalment number (1-indexed) |
| `due_date` | `time.Time` | Date the instalment collection is due |
| `amount_pence` | `int64` | Base premium portion due for the instalment in pence |
| `fee_pence` | `int64` | Allocated fee portion due for the instalment in pence |
| `status` | `string` | Instalment state: `due`, `collected`, or `failed` |

## Dependencies

- **Configuration:** Initialized via `plans.NewStore(dsn)` in `cmd/billing/main.go` using the `DATABASE_URL` environment variable.
- **Domain Service:** Injected into `plans.NewService(store, policies, payments, identity)` in `internal/plans` to support [[entities/plans-service|plans.Service]] orchestration.
- **Models:** Consumes `model.Plan` and `model.Instalment` definitions from `internal/plans/model/plan.go` (see [[entities/plan]] and [[entities/instalment]]).
