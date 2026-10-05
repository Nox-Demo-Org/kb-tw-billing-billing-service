---
type: Component
title: Downstream Clients
description: billing-service interacts with external downstream services via dedicated HTTP client wrappers implemented in package internal/clients.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/entities/downstream-clients.md
tags:
- billing-service
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/clients/http.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/clients/policy.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/clients/payments.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: internal/clients/http.go:L1-L38 -->
<!-- anchor: internal/clients/policy.go:L1-L29 -->
<!-- anchor: internal/clients/payments.go:L1-L29 -->

# Downstream Clients

`billing-service` interacts with external downstream services via dedicated HTTP client wrappers implemented in package `internal/clients`. These clients handle REST communication with `policy-admin`, `payments-gateway`, and `customer-identity`.

---

## Responsibilities

The package defines generic HTTP helpers and three specialized client types:

### 1. HTTP Helpers (`internal/clients/http.go`)
- `getJSON[T any](ctx context.Context, c *http.Client, url string) (T, error)`: Performs an HTTP `GET` request, validates that the status code is below 300, and decodes the JSON payload into type `T`.
- `postJSON(ctx context.Context, c *http.Client, url string, body any) error`: Serializes a body struct to JSON, sets the `Content-Type: application/json` header, performs an HTTP `POST` request, and returns an error if the status code is 300 or greater.

### 2. Policy Client (`internal/clients/policy.go`)
Communicates with `policy-admin` to fetch policy details required for plan generation and fee calculation.

- **Client Definition:** `PolicyClient` with base URL and a standard `http.Client` configured with a 2-second timeout (`NewPolicyClient(base string)`).
- **Target Endpoint:** `GET /policies/{id}` (relative to base URL).
- **Data Model:**
  ```go
  type Policy struct {
      ID                 string    `json:"id"`
      CustomerID         string    `json:"customer_id"`
      AnnualPremiumPence int64     `json:"annual_premium_pence"`
      StartDate          time.Time `json:"start_date"`
      Status             string    `json:"status"`
  }
  ```
- **Method:** `Get(ctx context.Context, id string) (Policy, error)`.

### 3. Payments Client (`internal/clients/payments.go`)
Communicates with `payments-gateway` to trigger instalment collections.

- **Client Definition:** `PaymentsClient` with base URL and an `http.Client` configured with a 5-second timeout (`NewPaymentsClient(base string)`).
- **Target Endpoint:** `POST /collections` (relative to base URL).
- **Data Model:**
  ```go
  type CollectionRequest struct {
      PlanID        string `json:"plan_id"`
      Instalment    int    `json:"instalment"`
      AmountPence   int64  `json:"amount_pence"`
      Method        string `json:"method"` // card | direct_debit
      IdempotencyID string `json:"idempotency_id"`
  }
  ```
- **Method:** `Collect(ctx context.Context, r CollectionRequest) error`.

### 4. Identity Client (`internal/clients/identity.go`)
Communicates with `customer-identity` to retrieve customer profile data for fee notices and arrears correspondence.

- **Client Definition:** `IdentityClient` with base URL and an `http.Client` configured with a 2-second timeout (`NewIdentityClient(base string)`).
- **Target Endpoint:** `GET /customers/{id}` (relative to base URL).
- **Data Model:**
  ```go
  type Customer struct {
      ID            string    `json:"id"`
      Name          string    `json:"name"`
      Email         string    `json:"email"`
      Postcode      string    `json:"postcode"`
      CustomerSince time.Time `json:"customer_since"`
      TenureYears   int       `json:"tenure_years"`
  }
  ```
- **Method:** `Get(ctx context.Context, id string) (Customer, error)`.
  *Note:* As documented in the client, `billing-service` uses `Name` and `Email` for letters; `TenureYears` is decoded but currently unused.

---

## Dependencies

- **`entities/plan-service`**: Consumes `PolicyClient` during plan setup and schedule creation, and utilizes customer details.
- **[[concepts/arrears-and-collections]]**: Uses `PaymentsClient` to dispatch `CollectionRequest` payloads to `payments-gateway` when instalments become due.
- **Environment Configuration**: Base URLs for each client are configured via environment variables:
  - `POLICY_ADMIN_URL` for `PolicyClient`
  - `PAYMENTS_URL` for `PaymentsClient`
  - `CUSTOMER_IDENTITY_URL` for `IdentityClient`
- **Standard Library**:
  - `net/http` for HTTP transport and request execution.
  - `context` for request deadline and cancellation management.
  - `encoding/json` for payload serialization and deserialization.
