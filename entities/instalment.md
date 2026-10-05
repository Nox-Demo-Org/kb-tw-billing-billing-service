---
type: Component
title: Instalment
description: The Instalment entity represents a single scheduled payment obligation belonging to a PaymentPlan.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/instalment.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/model/plan.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/service.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: internal/plans/model/plan.go:L1-L33 -->
<!-- anchor: internal/plans/service.go:L1-L73 -->

# Instalment

The `Instalment` entity represents a single scheduled payment obligation belonging to a [[entities/plan|PaymentPlan]]. It defines the due date, the breakdown between premium and allocated instalment fee, and tracks the lifecycle status of each collection attempt.

## Responsibilities

- **Payment Tracking:** Represents individual collection units within a payment plan schedule, identified by plan ID and ordinal instalment number (`1` to `N`).
- **Fee and Premium Accounting:** Carries the exact total charge (`AmountPence`) and the embedded fee allocation (`FeePence`) for that cycle, calculated in pence.
- **Collection Status Lifecycle:** Tracks whether an instalment is awaiting collection, successfully collected, or failed (`due`, `collected`, `failed`).
- **Schedule Calculation:** Computed deterministically via `schedule(p model.Plan)` during plan generation, spacing instalments on monthly date offsets from the plan start date.

## Data Model

Defined in `internal/plans/model/plan.go` and persisted in the `instalments` table:

```go
type Instalment struct {
    PlanID      string    `json:"plan_id"`
    Number      int       `json:"number"`
    DueDate     time.Time `json:"due_date"`
    AmountPence int64     `json:"amount_pence"`
    FeePence    int64     `json:"fee_pence"`
    Status      string    `json:"status"` // due | collected | failed
}
```

### Field Definitions

| Field | Type | JSON Key | Description |
|---|---|---|---|
| `PlanID` | `string` | `plan_id` | Foreign key reference to the parent [[entities/plan|PaymentPlan]] (`id`). |
| `Number` | `int` | `number` | 1-based sequential index of the instalment (`1` for single-pay yearly; `1` to `12` for monthly). |
| `DueDate` | `time.Time` | `due_date` | The calendar date the collection is due (`StartDate.AddDate(0, i, 0)`). |
| `AmountPence` | `int64` | `amount_pence` | Total collection amount in integer pence (`(AnnualPremiumPence / Instalments) + FeePence`). |
| `FeePence` | `int64` | `fee_pence` | Portion of `AmountPence` representing the allocated instalment fee. |
| `Status` | `string` | `status` | Collection state: `"due"`, `"collected"`, or `"failed"`. |

## Status Values

- **`due`**: Initial status when the instalment schedule is generated upon plan creation.
- **`collected`**: Updated when payment collection succeeds (e.g., via [[ap:kb-tw-billing-payments-gateway/summaries/api-spec#payments-collection-succeeded|payments.collection.succeeded]]).
- **`failed`**: Marked when collection attempts fail; may trigger missed payment flows and plan arrears handling.

## Schedule and Fee Distribution

Instalments are constructed in `internal/plans/service.go` by `schedule(p model.Plan)`:

1. **Count:** Evaluated by `instalmentsFor(freq)`:
   - `model.Monthly` (`"monthly"`): 12 instalments.
   - `model.Yearly` (`"yearly"`): 1 instalment.
2. **Base Premium Share:** Integer division of `p.AnnualPremiumPence / int64(p.Instalments)`.
3. **Fee Allocation:** Determined per instalment slice by [[concepts/instalment-fee-calculation|fees.PerInstalment(p)]] (referencing [[decisions/0002-instalment-fee|ADR 0002]]).
4. **Amount:** `AmountPence = each + feeParts[i]`.
5. **Due Dates:** Sequentially computed as `p.StartDate.AddDate(0, i, 0)` for `i` from `0` to `p.Instalments - 1`.

## Dependencies

- **[[entities/plan|PaymentPlan]] (`model.Plan`):** Parent entity to which every instalment belongs.
- **[[entities/plans-service|plans.Service]]:** Orchestrates schedule generation (`schedule()`) and handles plan lifecycle transitions.
- **[[entities/store|plans.Store]]:** Persists instalment records in the `instalments` relational database table.
- **[[concepts/instalment-fee-calculation|fees]]:** Supplies `fees.PerInstalment(plan)` to compute `FeePence` distributions.
- **[[concepts/arrears-and-collections|Arrears & Collections Engine]]:** Manages transitions between `due`, `collected`, and `failed` states.
