---
type: Architecture Decision
title: 'ADR-0002: Instalment fee on monthly plans'
description: 'Key implementation rules established in internal/fees/instalment.go: - The fee applies only to plans with frequency model.Monthly; yearly plans are charged £0.'
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/decisions/0002-instalment-fee.md
tags:
- billing-service
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

# ADR-0002: Instalment fee on monthly plans

## Status
Accepted (2023-03)

## Context
Monthly payment plans carry higher operational costs than yearly plans due to processing twelve distinct collections, handling failed direct debits, and issuing arrears correspondence. Finance requested that these operational costs be recovered from customers who choose to pay monthly rather than annually.

## Decision
Charge an instalment fee of 6% of the annual premium (`FeeRate = 0.06`) on all monthly plans, capped at £38.00 per year (`AnnualCapPence = 3800`). 

Key implementation rules established in `internal/fees/instalment.go`:
- The fee applies only to plans with frequency `model.Monthly`; yearly plans are charged £0.
- The total annual fee is calculated once when the plan is created via `InstalmentFee(plan model.Plan)`.
- The annual fee is distributed evenly across all instalments via `PerInstalment(plan model.Plan)`, with any remainder from integer division added to the first instalment.
- See [[concepts/instalment-fee-calculation]] for detailed calculation logic and examples.

## Consequences
- **Uniform Application:** A single rule applies to all customers. No exemptions were agreed upon at implementation time (tenure, vulnerability, and staff policies are all charged identically).
- **Immutability on Active Schedules:** The fee is fixed at the time the [[entities/plan|payment plan]] is created. Any future changes to fee rules will only affect newly created plans and policy renewals; existing running [[entities/instalment|instalment schedules]] retain their original calculated fee across the [[concepts/plan-lifecycle|plan lifecycle]].
- **API and UI Transparency:** The total fee (`fee_pence`) and per-instalment allocations are exposed via `GET /v1/plans/{policyId}` (see [[summaries/api-spec]]), allowing [[ap:kb-tw-digital-customer-portal/summaries/api-spec#get-v1-plans-policyId|customer-portal (/v1/plans/{policyId})]] to present the fee breakdown clearly on the customer's Payments page.
