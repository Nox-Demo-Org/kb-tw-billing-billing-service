---
okf_version: '0.2'
title: billing-service
description: billing-service is a Go service responsible for managing payment plans, payment collection schedules, monthly instalment fees, and cancellation refunds for Tidewell Mutual insurance policies.
generated:
  at: '2026-10-05T12:39:38Z'
---

# billing-service

`billing-service` is a Go service responsible for managing payment plans, payment collection schedules, monthly instalment fees, and cancellation refunds for Tidewell Mutual insurance policies.

### Core Responsibilities
- **Payment Plans & Schedules:** Generates yearly (single payment) and monthly (12 instalments) payment schedules upon policy issuance (`policy.policy.issued`).
- **Instalment Fee Calculation:** Calculates and distributes the 6% monthly instalment fee (capped at £38/year) across instalments, putting any remainder on the first payment.
- **Proration & Refunds:** Computes pro-rata daily premium refunds when policies are cancelled (`policy.policy.cancelled`).
- **Collection & Arrears Events:** Publishes collection notices (`billing.instalment.due`) and flags missed collections (`billing.payment.missed`) after failed attempts.
- **External Integrations:** Communicates with `policy-admin`, `payments-gateway`, and `customer-identity` via HTTP REST clients, and uses Google Cloud Pub/Sub for asynchronous event flows.

### How to Run
Configure environment variables (`POLICY_ADMIN_URL`, `PAYMENTS_URL`, `CUSTOMER_IDENTITY_URL`, `DATABASE_URL`, `GCP_PROJECT`) and start the HTTP service:
```bash
go run cmd/billing/main.go
```

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 3 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 3 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 2 pages. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
