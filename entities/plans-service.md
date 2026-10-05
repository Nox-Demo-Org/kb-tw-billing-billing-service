---
type: Component
title: Plans Service
description: The plans.Service struct in internal/plans/service.go is the core domain orchestrator for payment plans and instalment schedules.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/plans-service.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/service.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/plans/proration.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/cmd/billing/main.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: internal/plans/service.go:L1-L73 -->
<!-- anchor: internal/plans/proration.go:L1-L18 -->
<!-- anchor: cmd/billing/main.go:L1-L30 -->

# Plans Service

The `plans.Service` struct in `internal/plans/service.go` is the core domain orchestrator for [[entities/plan|payment plans]] and [[entities/instalment|instalment schedules]]. It manages plan creation upon policy issuance, schedule generation, fee application, and plan closure with pro-rata refund calculations.

## Responsibilities

- **Plan & Schedule Generation:** Orchestrates the creation of new payment plans via `CreateFromPolicy` when triggered by policy issuance events or REST endpoints. Fetches policy terms from policy administration, determines instalment counts based on frequency (12 for monthly, 1 for yearly), calculates fees via [[concepts/instalment-fee-calculation|fee utilities]], and builds out the instalment schedule.
- **Schedule Generation (`schedule`):** Computes per-instalment due dates (spaced monthly using `StartDate.AddDate(0, i, 0)`), base premium distribution, and instalment fee allocations, setting initial instalment statuses to `"due"`.
- **Plan Termination & Refund Calculations (`Close`):** Closes existing payment plans when a policy is cancelled, invoking `Prorate` to compute the unearned premium refund before updating the store.
- **Proration Engine (`Prorate`):** Calculates exact day-count unearned premium refund amounts from policy cancellation dates to the end of the policy year (`internal/plans/proration.go`).

## Dependencies

The `Service` struct requires four primary dependencies injected via `NewService`:

| Dependency | Type | Description |
| :--- | :--- | :--- |
| `store` | `*plans.Store` | Database persistence layer for plans and instalments ([[entities/store]]). |
| `policies` | `*clients.PolicyClient` | HTTP client for `policy-admin` to fetch policy details via `GET /v1/policies/{id}` ([[entities/downstream-clients]]). |
| `payments` | `*clients.PaymentsClient` | HTTP client for `payments-gateway` collection management ([[entities/downstream-clients]]). |
| `identity` | `*clients.IdentityClient` | HTTP client for `customer-identity` customer data ([[entities/downstream-clients]]). |

Additionally, the service interacts with internal fee calculation helpers:
- `fees.InstalmentFee`: Computes total annual instalment fees based on plan premium and frequency (see [[decisions/0002-instalment-fee]] and [[concepts/instalment-fee-calculation]]).
- `fees.PerInstalment`: Allocates fee amounts across individual instalments.

## Core Workflows

### Plan Creation (`CreateFromPolicy`)

1. Retrieves policy details (annual premium, customer ID, start date) via `policies.Get(ctx, policyID)`.
2. Resolves total instalment count using `instalmentsFor(freq)`:
   - `model.Monthly` $\rightarrow$ 12 instalments
   - `model.Yearly` (or other) $\rightarrow$ 1 instalment
3. Initializes a `model.Plan` in `"active"` status and calculates `plan.FeePence` using `fees.InstalmentFee(plan)`.
4. Generates the `[]model.Instalment` collection using `schedule(plan)`.
5. Persists both plan and schedule atomically via `store.Save(ctx, plan, instalments)`.

```go
func (s *Service) CreateFromPolicy(ctx context.Context, policyID string, freq model.Frequency) (model.Plan, error)
```

### Plan Closure & Proration (`Close`)

1. Loads the active plan by policy ID using `store.ByPolicy(ctx, policyID)`.
2. Computes the refundable pence amount using `Prorate(plan.AnnualPremiumPence, plan.StartDate, cancelledOn)`.
3. Calls `store.Close(ctx, plan.ID, refund)` to persist the cancellation and refund amount.

```go
func (s *Service) Close(ctx context.Context, policyID string, cancelledOn time.Time) error
```

### Daily Proration Calculation (`Prorate`)

Defined in `internal/plans/proration.go`, `Prorate` calculates unused premium on an exact day-count basis:
- Defines the policy end date as `start.AddDate(1, 0, 0)` (accounting for 366 days in leap years).
- If `cancelledOn` is not before `end`, the refund is `0`.
- Divides unused duration by total duration and truncates to integer pence:
  $$\text{refund} = \left\lfloor \text{annualPence} \times \frac{\text{unused days}}{\text{total days}} \right\rfloor$$

*(Note: Unlike `rating-service`, which rounds half up in its `ProRata.kt` implementation, `billing-service` truncates to integer).*

## Service Wiring

`plans.Service` is initialized in `cmd/billing/main.go` and shared across both the HTTP REST API router (`api.Mount`) and the Cloud Pub/Sub event subscriber (`events.Subscribe`):

- **Event Consumption:** Consumes `policy.policy.issued` and `policy.policy.cancelled` (see [[summaries/events]] and [[concepts/plan-lifecycle]]).
- **REST Endpoints:** Backs HTTP handlers such as `GET /v1/plans/{policyId}` and `POST /v1/plans` (see [[summaries/api-spec]]).
- **Separation of Concerns:** Implements architectural boundary separating payment plan management from policy administration (see [[decisions/0001-billing-owns-plans]]).
