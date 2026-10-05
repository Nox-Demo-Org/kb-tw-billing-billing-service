# Components and data models

One page per significant component and core data model.

## Pages

- [Downstream Clients](/entities/downstream-clients.md) — billing-service interacts with external downstream services via dedicated HTTP client wrappers implemented in package internal/clients.
- [Plan Data Model & Persistence](/entities/plan-model.md) — The plan-model entity defines the core domain models and database persistence layer for managing payment plans and scheduled instalments in billing-service.
- [Plan Service](/entities/plan-service.md) — The plans.Service struct in internal/plans/service.go encapsulates the core business logic for managing payment plans, generating payment schedules, and closing plans upon policy cancellation.
