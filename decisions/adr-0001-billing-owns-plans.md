---
type: Architecture Decision
title: 'ADR-0001: Billing owns payment plans'
description: 'Under this boundary: - policy-admin retains ownership of policy details and core premium amounts.'
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/decisions/adr-0001-billing-owns-plans.md
tags:
- billing-service
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

# ADR-0001: Billing owns payment plans

## Status
Accepted (2022-11)

## Context
Payment plans and instalment handling were previously managed within `policy-admin`. Managing premium payment schedules, instalment breakdowns, collection lifecycles, and payment tracking required a dedicated domain boundary separate from core policy administration.

## Decision
Payment plans and instalments are moved out of `policy-admin` into `billing-service` (tracked via `TWPOL-12`).

Under this boundary:
- `policy-admin` retains ownership of policy details and core premium amounts.
- `billing-service` owns how and when the premium is paid, including the [[entities/plan-model|payment_plans and instalments]] persistence tables.
- The two services never access or mutate each other's database tables directly.
- Communication between `policy-admin` and `billing-service` occurs via asynchronous Pub/Sub events:
  - `policy.policy.issued` (consumed by `billing-service` to initialize plans; see [[summaries/events-spec]])
  - `policy.policy.cancelled` (consumed by `billing-service` to close plans and calculate refunds; see [[concepts/proration-and-refunds]])
  - `billing.payment.missed` (published by `billing-service` when payment collection fails; see [[concepts/arrears-and-collections]])

## Consequences
- **Decoupled Persistence:** `billing-service` maintains its own dedicated database tables (`payment_plans` and `instalments`) without shared database access with `policy-admin`.
- **Domain Specialization:** Core billing capabilities—such as payment plan generation in [[entities/plan-service]], [[decisions/adr-0002-instalment-fee|instalment fee calculations]], and collection tracking—are managed independently within `billing-service`.
- **Event-Driven Integration:** System integration between policy management and billing operations relies on strict message contracts (`policy.policy.issued`, `policy.policy.cancelled`, and `billing.payment.missed`) as documented in [[summaries/events-spec]].
