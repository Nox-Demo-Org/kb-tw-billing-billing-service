---
type: Concept
title: Proration and Refunds
description: billing-service calculates pro-rata premium refunds when a policy is cancelled mid-term.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/concepts/proration-and-refunds.md
tags:
- billing-service
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

# Proration and Refunds

`billing-service` calculates pro-rata premium refunds when a policy is cancelled mid-term. The calculation determines the unused portion of the annual premium from the cancellation date until the end of the policy year.

---

## Proration Algorithm

Proration logic is implemented in `internal/plans/proration.go` via the `Prorate` function:

```go
func Prorate(annualPence int64, start, cancelledOn time.Time) int64
```

### Calculation Rules

1. **Policy Year End Date:** Calculated by adding exactly one year to the start date (`start.AddDate(1, 0, 0)`).
2. **Boundary Check:** If `cancelledOn` is on or after the end date (`!cancelledOn.Before(end)`), the refund is `0`.
3. **Exact Day Counting:** 
   - `total = end.Sub(start).Hours() / 24`
   - `unused = end.Sub(cancelledOn).Hours() / 24`
   - Leap years are handled naturally by the underlying timestamp durations (yielding 366 days).
4. **Formula and Rounding:**
   $$\text{refund} = \text{int64}\left(\text{float64}(\text{annualPence}) \times \frac{\text{unused}}{\text{total}}\right)$$
   `billing-service` uses integer truncation (casting the `float64` directly to `int64`).

---

## Cancellation Workflow

When a policy cancellation event (`policy.policy.cancelled`) is consumed (see [[summaries/events-spec]]):

1. [[entities/plan-service]] invokes `Service.Close(ctx, policyID, cancelledOn)`.
2. The active [[entities/plan-model|Plan]] is retrieved from the store using `store.ByPolicy(ctx, policyID)`.
3. `Prorate(plan.AnnualPremiumPence, plan.StartDate, cancelledOn)` calculates the refund amount in pence.
4. `store.Close(ctx, plan.ID, refund)` marks the plan as closed and persists the calculated refund.

```go
// Close ends a plan on policy.policy.cancelled and works out the refund with Prorate.
func (s *Service) Close(ctx context.Context, policyID string, cancelledOn time.Time) error {
	plan, err := s.store.ByPolicy(ctx, policyID)
	if err != nil {
		return err
	}
	refund := Prorate(plan.AnnualPremiumPence, plan.StartDate, cancelledOn)
	return s.store.Close(ctx, plan.ID, refund)
}
```

---

## Differences with `rating-service`

There is a known divergence between `billing-service` and `rating-service` regarding pro-rata calculations (tracked in Jira task **TWPOL-8**):

| Service | File / Implementation | Rounding Strategy | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`billing-service`** | `internal/plans/proration.go` (`Prorate`) | **Truncation** (`int64(...)`) | Cancellation refunds |
| **`rating-service`** | `ProRata.kt` | **Round Half Up** | Mid-term adjustment pricing |

Because `rating-service` rounds half up while `billing-service` truncates, refund calculations on cancellations and mid-term adjustment quotes can differ by 1 penny for the exact same date range.
