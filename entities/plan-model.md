---
type: Component
title: Plan Data Model & Persistence
description: The plan-model entity defines the core domain models and database persistence layer for managing payment plans and scheduled instalments in billing-service.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/plan-model.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/model/plan.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/store.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

<!-- anchor: internal/plans/model/plan.go:L1-L33 -->
<!-- anchor: internal/plans/store.go:L1-L17 -->

# Plan Data Model & Persistence

The `plan-model` entity defines the core domain models and database persistence layer for managing payment plans and scheduled instalments in `billing-service`. It encapsulates payment structure definitions, collection status lifecycles, and database operations across the `payment_plans` and `instalments` database tables.

The model is utilized directly by [[entities/plan-service]] to orchestrate schedule generation, fee application, and policy cancellation updates.

---

## Data Models

The models are defined in `internal/plans/model/plan.go`.

### `model.Frequency`
Specifies the frequency with which a policy's premium is paid:
```go
type Frequency string

const (
    Monthly Frequency = "monthly"
    Yearly  Frequency = "yearly"
)
```

### `model.Plan`
Represents the overall payment plan for a policy, mapped to the `payment_plans` database table.

| Field | Type | JSON Tag | Description |
| :--- | :--- | :--- | :--- |
| `ID` | `string` | `id` | Unique identifier for the payment plan. |
| `PolicyID` | `string` | `policy_id` | Identifier of the associated policy. |
| `CustomerID` | `string` | `customer_id` | Identifier of the customer. |
| `Frequency` | `Frequency` | `frequency` | Payment frequency (`monthly` or `yearly`). |
| `AnnualPremiumPence` | `int64` | `annual_premium_pence` | Total annual policy premium in pence. |
| `Instalments` | `int` | `instalments` | Number of scheduled instalments (e.g., 1 or 12). |
| `FeePence` | `int64` | `fee_pence` | Total instalment fee charged in pence (see [[concepts/instalment-fee-calculation]]). |
| `StartDate` | `time.Time` | `start_date` | Date the payment plan begins. |
| `Status` | `string` | `status` | Plan lifecycle status: `active`, `in_arrears`, or `closed`. |

### `model.Instalment`
Represents a single collection event within a plan, mapped to the `instalments` database table.

| Field | Type | JSON Tag | Description |
| :--- | :--- | :--- | :--- |
| `PlanID` | `string` | `plan_id` | Foreign identifier linking to the associated `Plan`. |
| `Number` | `int` | `number` | The instalment sequence number (e.g., 1 to 12). |
| `DueDate` | `time.Time` | `due_date` | Date when the payment is due for collection. |
| `AmountPence` | `int64` | `amount_pence` | Base premium amount due for this instalment in pence. |
| `FeePence` | `int64` | `fee_pence` | Portion of the instalment fee due for this instalment in pence. |
| `Status` | `string` | `status` | Collection status: `due`, `collected`, or `failed`. |

---

## Persistence Store

The persistence store is defined in `internal/plans/store.go` as `plans.Store`. It manages read and write operations targeting the underlying database tables `payment_plans` and `instalments`.

```go
type Store struct {
    dsn string
}

func NewStore(dsn string) *Store
```

### Store Methods

- `Save(ctx context.Context, p model.Plan, sched []model.Instalment) error`  
  Persists a newly generated `Plan` record and its associated collection schedule (`[]Instalment`).
- `ByPolicy(ctx context.Context, policyID string) (model.Plan, error)`  
  Fetches the payment plan corresponding to a given `policyID`.
- `Close(ctx context.Context, planID string, refundPence int64) error`  
  Closes an active plan when a policy is cancelled, recording the refunded premium amount (see [[concepts/proration-and-refunds]]).
- `MarkCollected(ctx context.Context, planID string, number int) error`  
  Updates the status of a specific instalment number under a plan to indicate successful collection (see [[concepts/arrears-and-collections]]).

---

## Responsibilities

- **Domain Representation:** Represents the schema and lifecycle statuses for payment plans (`active`, `in_arrears`, `closed`) and individual instalments (`due`, `collected`, `failed`).
- **Plan & Schedule Persistence:** Manages database writes and reads for the `payment_plans` and `instalments` tables.
- **Collection State Tracking:** Updates instalment collection records upon payment confirmation.
- **Plan Termination:** Updates plan status to `closed` along with refund adjustments when policies are terminated.

---

## Dependencies

- **Go Standard Library:**
  - `context.Context` for cancellation and execution deadlines in store operations.
  - `time.Time` for schedule dates and payment plan start dates.
- **Used by:**
  - [[entities/plan-service]]: Uses `Store` and `model` types to execute business logic for creating plans, tracking instalments, and closing plans.
  - [[summaries/api-spec]]: Uses the data model structures to serialize responses for plan retrieval endpoints.
  - [[decisions/adr-0001-billing-owns-plans]]: Grounded in the architectural decision establishing `billing-service` as the system of record for payment plans.
