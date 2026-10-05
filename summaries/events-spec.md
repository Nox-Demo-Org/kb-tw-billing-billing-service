---
type: Interface Reference
title: Events Specification
description: billing-service integrates asynchronously with other Tidewell Mutual services using Google Cloud Pub/Sub topics.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/summaries/events-spec.md
tags:
- billing-service
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/events/publisher.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/internal/events/subscriptions.go
- resource: https://github.com/Nox-Demo-Org/billing-service/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:39:38Z'
---

<!-- anchor: internal/events/publisher.go:L1-L24 -->
<!-- anchor: internal/events/subscriptions.go:L1-L24 -->
<!-- anchor: README.md:L1-L40 -->

# Events Specification

`billing-service` integrates asynchronously with other Tidewell Mutual services using Google Cloud Pub/Sub topics. Event definitions and subscriptions are managed in `internal/events/publisher.go` and `internal/events/subscriptions.go`.

---

## Published Events

### `billing.instalment.due`
- **Topic Constant**: `TopicInstalmentDue = "billing.instalment.due"`
- **Payload Struct**: `events.InstalmentDue`
- **Timing / Trigger**: Published 5 days before each instalment collection date.
- **Subscribers**: `notifications-hub` (to dispatch notices) and `payments-gateway` (to trigger payment processing).
- **See Also**: [[concepts/arrears-and-collections]], [[entities/plan-service]]

#### Schema
```json
{
  "plan_id": "string",
  "policy_id": "string",
  "customer_id": "string",
  "instalment": 1,
  "amount_pence": 8333,
  "fee_pence": 316,
  "due_date": "2024-05-01"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `plan_id` | `string` | Unique identifier of the payment plan. |
| `policy_id` | `string` | Associated policy ID. |
| `customer_id` | `string` | Customer identifier associated with the policy. |
| `instalment` | `int` | Instalment sequence number (e.g., `1` to `12`). |
| `amount_pence` | `int64` | Base instalment amount in pence. |
| `fee_pence` | `int64` | Monthly instalment fee allocated to this instalment in pence. |
| `due_date` | `string` | Collection due date. |

---

### `billing.payment.missed`
- **Topic Constant**: `TopicPaymentMissed = "billing.payment.missed"`
- **Payload Struct**: `events.PaymentMissed`
- **Timing / Trigger**: Published when an instalment collection attempt fails twice.
- **Subscribers**: `policy-admin` (for policy status tracking) and `notifications-hub` (for arrears letters).
- **See Also**: [[concepts/arrears-and-collections]]

#### Schema
```json
{
  "plan_id": "string",
  "policy_id": "string",
  "instalment": 1,
  "attempts": 2
}
```

| Field | Type | Description |
| --- | --- | --- |
| `plan_id` | `string` | Unique identifier of the payment plan. |
| `policy_id` | `string` | Associated policy ID. |
| `instalment` | `int` | Instalment sequence number that failed. |
| `attempts` | `int` | Number of failed collection attempts (e.g., `2`). |

---

## Consumed Events

Subscriptions are wired in `events.Subscribe(project string, svc *plans.Service, pub *Publisher)`.

| Topic Constant | Topic Name | Subscription Name | Handler / Target Action | Source Service |
| --- | --- | --- | --- | --- |
| `TopicPolicyIssued` | `policy.policy.issued` | `billing-policy-issued` | Calls `svc.CreateFromPolicy` to create a plan and schedule instalments | `policy-admin` |
| `TopicPolicyCancelled` | `policy.policy.cancelled` | `billing-policy-cancelled` | Calls `svc.Close` to close the plan and calculate refunds | `policy-admin` |
| `TopicCollectionSucceeded` | `payments.collection.succeeded` | `billing-collection-succeeded` | Calls `store.MarkCollected` to mark the instalment as paid | `payments-gateway` |

### Explicitly Excluded Events
- `customer.profile.updated`: `billing-service` does **not** subscribe to customer address updates. Billing reads the customer address once via [[entities/downstream-clients]] at the time the plan is created, ensuring fee and arrears notices are addressed according to the customer's state at policy inception.

---

## Related Documentation
- [[summaries/api-spec]] — Synchronous REST API endpoints.
- [[entities/plan-service]] — Core service orchestration for plans and schedules.
- [[concepts/arrears-and-collections]] — Instalment due schedules and missed collection handling.
- [[concepts/proration-and-refunds]] — Actions triggered when `policy.policy.cancelled` is received.
- [[index]] — Architecture and integration overview.
