---
okf_version: '0.2'
title: billing-service
description: billing-service manages payment plans, instalment schedules, instalment fees, collection workflows, and mid-term cancellation refunds for Tidewell Mutual insurance policies.
generated:
  at: '2026-10-05T13:11:32Z'
---

# billing-service

`billing-service` manages payment plans, instalment schedules, instalment fees, collection workflows, and mid-term cancellation refunds for Tidewell Mutual insurance policies.

### Core Purpose & Responsibilities
- **Payment Plans & Schedules:** Maintains payment schedules for insurance policies, supporting both yearly (single payment) and monthly (12 instalments) frequencies.
- **Fee Engine:** Calculates and allocates instalment fees on monthly plans (6% of annual premium, capped at £38/year, front-loaded for remainders).
- **Lifecycle & Events:** Creates plans when policies are issued (`policy.policy.issued`), triggers collection schedules (`billing.instalment.due`), publishes failure alerts on repeated collection issues (`billing.payment.missed`), and calculates daily pro-rata refunds when policies are cancelled (`policy.policy.cancelled`).
- **Integrations:** Communicates with `policy-admin` for policy details, `payments-gateway` for collections, and `customer-identity` for customer profiles.

### Running the Service
The service is written in Go and exposes an HTTP REST API on port `:8080`. Environment variables required at startup:
- `POLICY_ADMIN_URL`: Base URL for policy admin service (e.g. `http://policy-admin/v1`)
- `PAYMENTS_URL`: Base URL for payments gateway (e.g. `http://payments-gateway/v1`)
- `CUSTOMER_IDENTITY_URL`: Base URL for customer identity service (e.g. `http://customer-identity/v1`)
- `DATABASE_URL`: Connection string for PostgreSQL / relational storage
- `GCP_PROJECT`: Project identifier for Google Cloud Pub/Sub subscriptions and publishers

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 4 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 5 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 2 pages. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
