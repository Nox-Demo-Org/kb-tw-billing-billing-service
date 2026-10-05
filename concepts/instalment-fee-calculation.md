---
type: Concept
title: Instalment Fee Calculation
description: billing-service calculates and distributes an instalment fee for monthly payment plans to recover the operational costs of managing ongoing monthly collections, Direct Debit processing, and arrears workflows.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/concepts/instalment-fee-calculation.md
tags:
- billing-service
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

# Instalment Fee Calculation

`billing-service` calculates and distributes an instalment fee for monthly payment plans to recover the operational costs of managing ongoing monthly collections, Direct Debit processing, and arrears workflows.

The fee calculation and distribution logic is implemented in `internal/fees/instalment.go` as established in [[decisions/adr-0002-instalment-fee]].

---

## Core Fee Parameters

| Parameter | Code Constant | Value | Description |
| :--- | :--- | :--- | :--- |
| **Fee Rate** | `FeeRate` | `0.06` (6%) | Percentage of the annual premium charged for monthly payment plans. |
| **Annual Cap** | `AnnualCapPence` | `3800` (£38.00) | Maximum total fee charged over the course of a policy year. |

---

## Fee Calculation Rules

The annual fee is computed by `InstalmentFee(plan model.Plan) int64` according to the following rules:

1. **Payment Frequency Check:** 
   - If `plan.Frequency != model.Monthly` (e.g., `model.Yearly`), the fee is always `0`.
2. **Rate Application:**
   - For monthly plans, the fee is calculated as `6%` of `plan.AnnualPremiumPence`:
     $$\text{fee} = \lfloor \text{AnnualPremiumPence} \times 0.06 \rfloor$$
3. **Cap Enforcement:**
   - If the calculated fee exceeds `3800` pence (£38.00), the fee is clamped to `3800` pence (`AnnualCapPence`).

```go
func InstalmentFee(plan model.Plan) int64 {
	if plan.Frequency != model.Monthly {
		return 0
	}
	fee := int64(float64(plan.AnnualPremiumPence) * FeeRate)
	if fee > AnnualCapPence {
		fee = AnnualCapPence
	}
	return fee
}
```

### Exemptions
Currently, **there are no fee exemptions**. All monthly plans pay the standard fee regardless of customer tenure, vulnerability flags, or staff policy designations.

---

## Remainder Distribution Across Instalments

When dividing the total annual fee across individual instalments, integer division can result in rounding remainders. The `PerInstalment(plan model.Plan) []int64` function distributes the fee evenly across all instalments and allocates any leftover pence to the **first instalment**:

1. Calculate base instalment fee:
   $$\text{each} = \lfloor \frac{\text{fee}}{\text{Instalments}} \rfloor$$
2. Allocate `each` to every instalment slot $0 \dots (N-1)$.
3. Add the remainder to the first instalment ($i = 0$):
   $$\text{remainder} = \text{fee} - (\text{each} \times \text{Instalments})$$
   $$\text{Instalment}[0] = \text{each} + \text{remainder}$$

```go
func PerInstalment(plan model.Plan) []int64 {
	fee := InstalmentFee(plan)
	out := make([]int64, plan.Instalments)
	if plan.Instalments == 0 {
		return out
	}
	each := fee / int64(plan.Instalments)
	for i := range out {
		out[i] = each
	}
	out[0] += fee - each*int64(plan.Instalments)
	return out
}
```

### Example Breakdown
For a monthly plan with 12 instalments and an annual premium of £500.00 (`50000` pence):
- **Total Fee:** $50000 \times 0.06 = 3000\text{ pence}$ (£30.00, below cap).
- **Base Fee per Instalment:** $3000 / 12 = 250\text{ pence}$ (£2.50).
- **Remainder:** $3000 - (250 \times 12) = 0\text{ pence}$.
- **Schedule:** 12 instalments of 250 pence each.

For a monthly plan where division produces a remainder (e.g., total fee of 3800 pence across 12 instalments):
- **Base Fee per Instalment:** $3800 / 12 = 316\text{ pence}$.
- **Remainder:** $3800 - (316 \times 12) = 3800 - 3792 = 8\text{ pence}$.
- **Instalment 1:** $316 + 8 = 324\text{ pence}$ (£3.24).
- **Instalments 2–12:** 316 pence each (£3.16).
- **Sum Total:** $324 + (11 \times 316) = 3800\text{ pence}$.

---

## Plan Lifecycle and API Exposure

- **Immutability on Active Plans:** The fee is calculated and locked once when the plan is generated (upon receiving `policy.policy.issued` via [[entities/plan-service]]). Changes to fee rules only impact new plans and policy renewals; active schedules retain their original fee structure.
- **REST Visibility:** The fee appears as a dedicated field (`fee_pence`) on `GET /v1/plans/{policyId}` so client applications (such as the customer portal) can display it separately from the base policy premium (see [[summaries/api-spec]]).
- **Database Storage:** The total fee and per-instalment fee breakdowns are persisted in the `payment_plans` and `instalments` tables (see [[entities/plan-model]]).
- **Cancellations:** When a policy is cancelled, cancellation refunds are computed pro-rata to the day across total amounts paid including fees (see [[concepts/proration-and-refunds]]).

---

## Operational and Retention Context

Internal retention reviews (Confluence `TWBILL`) show:
- ~58% of customers pay monthly.
- Monthly customers with >5 years tenure cancel at 2.1x the normal rate after fee/price increases.
- While `customer-identity` exposes a `tenure_years` field (see [[entities/downstream-clients]]), `billing-service` does not currently consume tenure to waive instalment fees.
