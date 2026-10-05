---
type: Interface Reference
title: Pub/Sub Message Schemas and Events
description: billing-service relies on Google Cloud Pub/Sub for asynchronous event-driven workflows across the policy lifecycle, collection orchestration, and failure notification.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/summaries/events.md
tags:
- billing-service
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/events/subscriptions.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/events/publisher.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

<!-- anchor: internal/events/subscriptions.go:L1-L24 -->
<!-- anchor: internal/events/publisher.go:L1-L24 -->
<!-- anchor: README.md:L1-L40 -->

# Pub/Sub Message Schemas and Events

`billing-service` relies on Google Cloud Pub/Sub for asynchronous event-driven workflows across the policy lifecycle, collection orchestration, and failure notification.

Event wiring and subscription orchestration live in `internal/events/subscriptions.go`, while message publishing structures are defined in `internal/events/publisher.go`.

---

## Published Topics

`billing-service` publishes events using the `Publisher` struct (`internal/events/publisher.go`), which is initialized with `GCP_PROJECT`.

### 1. `billing.instalment.due`
* **Topic Constant:** `TopicInstalmentDue` (`"billing.instalment.due"`)
* **When Published:** Published 5 days before each scheduled instalment collection date (see [[concepts/arrears-and-collections]] and [[entities/plans-service]]).
* **Consumers:** `payments-gateway` (triggers payment processing) and `notifications-hub` (sends advance payment notices).
* **Payload Structure (`InstalmentDue`):**

```go
type InstalmentDue struct {
    PlanID      string `json:"plan_id"`
    PolicyID    string `json:"policy_id"`
    CustomerID  string `json:"customer_id"`
    Instalment  int    `json:"instalment"`
    AmountPence int64  `json:"amount_pence"`
    FeePence    int64  `json:"fee_pence"`
    DueDate     string `json:"due_date"`
}
```

| Field | Type | Description |
| --- | --- | --- |
| `plan_id` | `string` | Unique identifier of the payment plan (`PaymentPlan.ID`). |
| `policy_id` | `string` | Identifier of the underlying insurance policy. |
| `customer_id` | `string` | Identifier of the policyholder. |
| `instalment` | `int` | Sequence number of the instalment (e.g., `1` to `12`). |
| `amount_pence` | `int64` | Total amount to be collected for this instalment in pence (including fee). |
| `fee_pence` | `int64` | Proportion of the instalment fee allocated to this instalment in pence. |
| `due_date` | `string` | Date collection is due (ISO format `YYYY-MM-DD`). |

---

### 2. `billing.payment.missed`
* **Topic Constant:** `TopicPaymentMissed` (`"billing.payment.missed"`)
* **When Published:** Published when an instalment collection attempt fails twice (initial attempt fails, `payments-gateway` retries after 3 working days, and the retry also fails).
* **Consumers:** `policy-admin` (initiates the 14-day cancellation letter workflow) and `notifications-hub` (sends an SMS notification to the customer).
* **Payload Structure (`PaymentMissed`):**

```go
type PaymentMissed struct {
    PlanID     string `json:"plan_id"`
    PolicyID   string `json:"policy_id"`
    Instalment int    `json:"instalment"`
    Attempts   int    `json:"attempts"`
}
```

| Field | Type | Description |
| --- | --- | --- |
| `plan_id` | `string` | Unique identifier of the payment plan. |
| `policy_id` | `string` | Identifier of the policy associated with the missed payment. |
| `instalment` | `int` | Instalment sequence number that failed. |
| `attempts` | `int` | Total number of collection attempts made (typically `2`). |

---

## Subscribed Topics

Subscriptions are registered in `Subscribe(project string, svc *plans.Service, pub *Publisher)` in `internal/events/subscriptions.go`.

| Subscription Name | Subscribed Topic | Source Service | Handler Target | Purpose |
| --- | --- | --- | --- | --- |
| `billing-policy-issued` | `policy.policy.issued` (`TopicPolicyIssued`) | `policy-admin` | `svc.CreateFromPolicy` | Triggered when a new policy is issued. Creates a [[entities/plan|PaymentPlan]], calculates [[concepts/instalment-fee-calculation|instalment fees]], and generates the [[entities/instalment|Instalment]] schedule. See [[concepts/plan-lifecycle]]. |
| `billing-policy-cancelled` | `policy.policy.cancelled` (`TopicPolicyCancelled`) | `policy-admin` | `svc.Close` | Triggered when a policy is cancelled mid-term. Calculates daily [[concepts/proration-and-refunds|pro-rata refunds]] and closes the plan. |
| `billing-collection-succeeded` | [[ap:kb-tw-billing-payments-gateway/summaries/api-spec#payments-collection-succeeded|payments-gateway (payments.collection.succeeded)]] (`TopicCollectionSucceeded`) | `payments-gateway` (see [[ap:kb-tw-billing-payments-gateway/summaries/api-spec#payments-collection-succeeded|payments-gateway (payments.collection.succeeded)]]) | `store.MarkCollected` | Marks the specific instalment record as collected in the [[entities/store|persistence layer]]. |

---

## Explicitly Unsubscribed Events

### [[ap:kb-tw-customer-platform-customer-identity/summaries/api-spec#customer-profile-updated|customer-identity (customer.profile.updated)]]
* **Topic:** [[ap:kb-tw-customer-platform-customer-identity/summaries/api-spec#customer-profile-updated|customer-identity (customer.profile.updated)]] (published by [[ap:kb-tw-customer-platform-customer-identity/summaries/api-spec#customer-profile-updated|customer-identity]])
* **Status:** **Not subscribed.**
* **Architectural Rationale:** `billing-service` fetches customer contact and address details via REST (`GET /v1/customers/{id}`) only once when the payment plan is initially created. Fee schedules and arrears notices rely on the snapshot address captured at plan inception.
