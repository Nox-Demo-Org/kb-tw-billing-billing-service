---
type: Architecture Decision
title: 'ADR-0001: Billing Owns Payment Plans'
description: 'Under this architecture: - policy-admin remains the source of truth for the policy and its annual premium.'
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/decisions/0001-billing-owns-plans.md
tags:
- billing-service
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

# ADR-0001: Billing Owns Payment Plans

## Status
Accepted (2022-11)

## Context
Historically, payment plans and instalment schedules were managed inside `policy-admin`. Coupling policy administration with billing operations created tight dependencies between underwriting/policy lifecycle management and financial collection orchestration.

## Decision
Payment plans and instalments are moved out of `policy-admin` into [[index|billing-service]] (tracked under Jira issue `TWPOL-12`).

Under this architecture:
- `policy-admin` remains the source of truth for the policy and its annual premium.
- `billing-service` owns how and when the premium is paid, managing [[entities/plan|payment plans]], [[entities/instalment|instalment schedules]], and [[concepts/instalment-fee-calculation|fee calculations]].
- The two services interact strictly through asynchronous domain events and REST APIs, never by reading or writing to each other's database tables directly.

### Inter-Service Communication
The boundary between `policy-admin` and `billing-service` is maintained through specific events:
- **`policy.policy.issued`**: Consumed by `billing-service` when a policy is issued to trigger initial plan and schedule generation (see [[concepts/plan-lifecycle]] and [[summaries/events]]).
- **`policy.policy.cancelled`**: Consumed by `billing-service` to terminate active payment plans and calculate pro-rata refunds (see [[concepts/proration-and-refunds]]).
- **`billing.payment.missed`**: Published by `billing-service` when collection attempts fail repeatedly to alert downstream systems (see [[concepts/arrears-and-collections]]).

Direct data retrieval relies on REST endpoints such as `GET /v1/policies/{id}` via [[entities/downstream-clients]].

## Consequences
- **Dedicated Persistence**: `billing-service` maintains its own relational storage schema (`payment_plans` and `instalments` tables via [[entities/store]]).
- **Decoupled Lifecycles**: Policy administration changes can occur independently of collection workflows and payment frequency models.
- **Asynchronous Integration**: The service relies on message delivery and idempotent event processing to keep policy state and billing schedules synchronized.
