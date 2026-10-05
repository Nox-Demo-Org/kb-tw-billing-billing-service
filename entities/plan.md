---
type: Component
title: Payment Plan Entity
description: The Plan domain entity (also referred to as PaymentPlan) represents the financial payment plan for a policy in billing-service.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/plan.md
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

# Payment Plan Entity

The `Plan` domain entity (also referred to as `PaymentPlan`) represents the financial payment plan for a policy in `billing-service`. It defines the payment cadence, total annual premium, number of instalments, associated instalment fees, and lifecycle state.

In the database, plans map to the `payment_plans` table (see [[entities/store]]).

## Responsibilities

- **Payment Cadence & Breakdown:** Captures whether a policy is billed annually or in monthly instalments.
- **Premium & Fee Tracking:** Retains the policy's `AnnualPremiumPence` and the calculated instalment fee `FeePence` in integer pence.
- **Schedule Foundation:** Serves as the parent entity for generating individual [[entities/instalment]] records.
- **Plan Status Tracking:** Maintains the current operational billing state (`active`, `in_arrears`, `closed`).

## Dependencies

- **[[entities/plans-service]]:** Coordinates creation of plans via `CreateFromPolicy` and plan termination via `Close`.
- **[[entities/instalment]]:** Child schedule records associated via `PlanID`.
- **[[concepts/instalment-fee-calculation]]:** Calculates `FeePence` using `fees.InstalmentFee` and splits fee allocations across instalments via `fees.PerInstalment`.
- **[[concepts/proration-and-refunds]]:** Calculates pro-rata refund amounts on policy cancellation before closing the plan.
- **[[entities/store]]:** Handles persistence to the `payment_plans` table.
- **[[entities/downstream-clients]]:** Source policy details (`AnnualPremiumPence`, `CustomerID`, `StartDate`) are fetched from `policy-admin` via `clients.PolicyClient`.

## Model Definition

Defined in `internal/plans/model/plan.go`:

```go
type Frequency string

const (
    Monthly Frequency = "monthly"
    Yearly  Frequency = "yearly"
)

type Plan struct {
    ID                 string    `json:"id"`
    PolicyID           string    `json:"policy_id"`
    CustomerID         string    `json:"customer_id"`
    Frequency          Frequency `json:"frequency"`
    AnnualPremiumPence int64     `json:"annual_premium_pence"`
    Instalments        int       `json:"instalments"`
    FeePence           int64     `json:"fee_pence"`
    StartDate          time.Time `json:"start_date"`
    Status             string    `json:"status"` // active | in_arrears | closed
}
```

### Fields

| Field | Type | JSON Key | Description |
|---|---|---|---|
| `ID` | `string` | `id` | Unique identifier for the payment plan. |
| `PolicyID` | `string` | `policy_id` | Identifier of the associated policy from `policy-admin`. |
| `CustomerID` | `string` | `customer_id` | Identifier of the customer owning the policy. |
| `Frequency` | `Frequency` | `frequency` | Payment frequency (`monthly` or `yearly`). |
| `AnnualPremiumPence` | `int64` | `annual_premium_pence` | Total annual policy premium expressed in integer pence. |
| `Instalments` | `int` | `instalments` | Number of scheduled payment instalments (12 for `monthly`, 1 for `yearly`). |
| `FeePence` | `int64` | `fee_pence` | Total instalment fee charged for the plan in integer pence (see [[concepts/instalment-fee-calculation]] and [[decisions/0002-instalment-fee]]). |
| `StartDate` | `time.Time` | `start_date` | Policy start date used to anchor instalment due dates. |
| `Status` | `string` | `status` | Operational status of the plan (`active`, `in_arrears`, `closed`). |

## Plan Statuses

A plan transitions through the following statuses during its lifecycle (see [[concepts/plan-lifecycle]] and [[concepts/arrears-and-collections]]):

- `active`: The plan is open, and scheduled collections proceed as planned.
- `in_arrears`: One or more instalments have failed collection and have not been resolved.
- `closed`: The plan has completed or was terminated early due to policy cancellation (`policy.policy.cancelled`).

## Schedule Models

When a plan is constructed in [[entities/plans-service]], `schedule(p model.Plan)` generates a slice of `model.Instalment` entities:

1. **Count Determination:**
   - `Monthly` frequency generates `12` instalments.
   - `Yearly` frequency generates `1` instalment.
2. **Base Amount Calculation:**
   - Base instalment premium = `p.AnnualPremiumPence / int64(p.Instalments)`.
3. **Fee Allocation:**
   - `fees.PerInstalment(p)` returns a slice of fee amounts in pence distributed across each instalment.
4. **Instalment Construction:**
   - `Number`: 1-based index (`1` to `p.Instalments`).
   - `DueDate`: `p.StartDate.AddDate(0, i, 0)` for monthly steps.
   - `AmountPence`: Base premium plus the fee portion (`each + feeParts[i]`).
   - `FeePence`: The fee portion allocated to this specific instalment.
   - `Status`: Initialized to `"due"`.
