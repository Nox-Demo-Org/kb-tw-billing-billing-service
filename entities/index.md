# Components and data models

One page per significant component and core data model.

## Pages

- [Downstream Clients](/entities/downstream-clients.md) — billing-service interacts with external downstream services via dedicated HTTP client wrappers implemented in package internal/clients.
- [Instalment](/entities/instalment.md) — The Instalment entity represents a single scheduled payment obligation belonging to a PaymentPlan.
- [Payment Plan Entity](/entities/plan.md) — The Plan domain entity (also referred to as PaymentPlan) represents the financial payment plan for a policy in billing-service.
- [Plans Service](/entities/plans-service.md) — The plans.Service struct in internal/plans/service.go is the core domain orchestrator for payment plans and instalment schedules.
- [Store](/entities/store.md) — Store (internal/plans/store.go) is the persistence layer responsible for reading and writing payment plans and instalment schedules to the relational database.
