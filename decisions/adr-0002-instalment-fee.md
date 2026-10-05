---
type: Architecture Decision
title: 'ADR-0002: Instalment Fee on Monthly Payment Plans'
description: Finance requested that the costs associated with managing monthly collections be recovered directly from customers who select monthly instalment schedules rather than absorbing them into baseline premiums.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/decisions/adr-0002-instalment-fee.md
tags:
- billing-service
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

# ADR-0002: Instalment Fee on Monthly Payment Plans

## Status
Accepted (2023-03)

## Context
Monthly payment plans incur higher operational and collection overhead compared to single annual upfront payments. Monthly processing requires twelve individual collection attempts, handling failed Direct Debits and card charges, and triggering arrears workflows and notification letters (see [[concepts/arrears-and-collections]]). 

Finance requested that the costs associated with managing monthly collections be recovered directly from customers who select monthly instalment schedules rather than absorbing them into baseline premiums.

## Decision
We implemented a standardized instalment fee rule in `internal/fees/instalment.go` applied during payment plan creation (see [[concepts/instalment-fee-calculation]]):

1. **Fee Calculation:**
   - Yearly plans (`model.Yearly`) incur no fee (`0` pence).
   - Monthly plans (`model.Monthly`) are charged 6% (`FeeRate = 0.06`) of the total annual premium (`plan.AnnualPremiumPence`).
   - The fee is capped at a maximum of £38.00 per year (`AnnualCapPence = 3800`).
2. **Distribution Across Instalments:**
   - The fee is calculated once at plan generation time by `InstalmentFee()`.
   - `PerInstalment()` divides the total fee equally across all instalments, allocating any division remainder to the first instalment (`out[0]`).
3. **Execution Point:**
   - Fee calculation is executed during plan creation in [[entities/plan-service]] upon consuming `policy.policy.issued` or through manual creation, storing values in [[entities/plan-model]].

```go
const (
    FeeRate        = 0.06
    AnnualCapPence = 3800
)
```

## Consequences
- **Uniform Application Across All Customers:** The fee applies to all monthly plans uniformly. There are no automated exemptions for customer vulnerability, staff policies, or customer tenure (even though `customer-identity` tracks `tenure_years`, [[entities/downstream-clients]] does not consume tenure for fee waivers).
- **Plan Immutability:** The instalment fee is locked in when the plan record is created in `payment_plans`. Changes to fee rules or caps only impact new plans and policy renewals; existing active instalment schedules maintain their originally calculated fee.
- **API Visibility:** The calculated fee is returned as an explicit field (`fee_pence`) and exposed within individual instalment breakdowns via `GET /v1/plans/{policyId}` (see [[summaries/api-spec]]), enabling the customer portal to present the fee breakdown clearly to policyholders.
- **Operational & Retention Impact:** According to retention analysis (Confluence `TWBILL: Retention and fees, FY26`), approximately 58% of customers pay monthly. Customers with tenure over 5 years pay an average of £38/year in fees and churn at 2.1 times the baseline rate following fee or price increases, leading to approximately 300 manual fee refunds per month managed by contact centre agents.
- **Cancellations & Proration:** When a policy is cancelled mid-term, premium and fee refunds are calculated pro-rata to the day across the active schedule (see [[concepts/proration-and-refunds]]).

## Related Decisions
- [[decisions/adr-0001-billing-owns-plans]] – Decoupled plan generation and fee calculation out of `policy-admin` into `billing-service`.
