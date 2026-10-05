---
type: Component
title: Plan Service
description: The plans.Service struct in internal/plans/service.go encapsulates the core business logic for managing payment plans, generating payment schedules, and closing plans upon policy cancellation.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/plan-service.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/service.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/cmd/billing/main.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

<!-- anchor: internal/plans/service.go:L1-L73 -->
<!-- anchor: cmd/billing/main.go:L1-L30 -->

# Plan Service

The `plans.Service` struct in `internal/plans/service.go` encapsulates the core business logic for managing payment plans, generating payment schedules, and closing plans upon policy cancellation. It is instantiated during application startup in `cmd/billing/main.go` and consumed by HTTP routes (see [[summaries/api-spec]]) and Pub/Sub event handlers (see [[summaries/events-spec]]).

## Responsibilities

The primary responsibilities of `plans.Service` include:

- **Plan Creation:** Implemented in `CreateFromPolicy(ctx, policyID, freq)`. When a `policy.policy.issued` event is received, the service retrieves the policy details from `policy-admin` via `clients.PolicyClient`, calculates any applicable instalment fees using `fees.InstalmentFee`, builds the schedule of instalments, and persists the new plan and schedule in the database via `Store.Save` (see [[entities/plan-model]] and [[decisions/adr-0001-billing-owns-plans]]).
- **Schedule Generation:** Implemented in the internal `schedule(p)` helper:
  - Determines the number of instalments via `instalmentsFor(freq)` (12 for `model.Monthly`, 1 for other frequencies such as yearly).
  - Divides `AnnualPremiumPence` evenly across instalments (`AnnualPremiumPence / int64(Instalments)`).
  - Retrieves distributed fee allocations from `fees.PerInstalment(p)` (see [[concepts/instalment-fee-calculation]] and [[decisions/adr-0002-instalment-fee]]).
  - Generates `model.Instalment` records with sequential due dates spaced monthly (`StartDate.AddDate(0, i, 0)`), combined amount (`each + feeParts[i]`), fee amount, and initial status `"due"`.
- **Plan Cancellation & Proration:** Implemented in `Close(ctx, policyID, cancelledOn)`. Triggered by `policy.policy.cancelled` events, it fetches the existing plan from `Store.ByPolicy`, calculates the refund amount using `Prorate(AnnualPremiumPence, StartDate, cancelledOn)`, and marks the plan as closed with the calculated refund via `Store.Close` (see [[concepts/proration-and-refunds]]).

## Dependencies

`plans.Service` is constructed via `NewService` with the following dependencies:

| Dependency | Type / Package | Purpose |
|---|---|---|
| `store` | `*plans.Store` | Persists and retrieves `model.Plan` and `model.Instalment` records (see [[entities/plan-model]]). |
| `policies` | `*clients.PolicyClient` | Downstream HTTP client for `policy-admin` (`GET /v1/policies/{id}`) to fetch policy premium, start date, and customer ID (see [[entities/downstream-clients]]). |
| `payments` | `*clients.PaymentsClient` | Downstream HTTP client for `payments-gateway` (see [[entities/downstream-clients]] and [[concepts/arrears-and-collections]]). |
| `identity` | `*clients.IdentityClient` | Downstream HTTP client for `customer-identity` (see [[entities/downstream-clients]]). |
| `fees` | `internal/fees` | Package providing `fees.InstalmentFee(plan)` and `fees.PerInstalment(plan)` (see [[concepts/instalment-fee-calculation]]). |

## Key Methods

### `CreateFromPolicy`
```go
func (s *Service) CreateFromPolicy(ctx context.Context, policyID string, freq model.Frequency) (model.Plan, error)
```
Fetches policy details from `s.policies.Get(ctx, policyID)`, sets initial status to `"active"`, computes `FeePence`, generates the instalment list, and saves the plan and its schedule.

### `Close`
```go
func (s *Service) Close(ctx context.Context, policyID string, cancelledOn time.Time) error
```
Retrieves the plan by policy ID, computes the refund amount using `Prorate`, and invokes `s.store.Close(ctx, plan.ID, refund)`.
