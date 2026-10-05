---
type: Concept
title: Plan Lifecycle
description: The payment plan lifecycle in billing-service governs the creation, scheduling, servicing, and termination of policy payment plans and their associated instalments.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-billing-service/blob/main/concepts/plan-lifecycle.md
tags:
- billing-service
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T13:11:32Z'
---

# Plan Lifecycle

The payment plan lifecycle in `billing-service` governs the creation, scheduling, servicing, and termination of policy payment plans and their associated instalments. 

The lifecycle transitions through distinct phases driven by domain events emitted by `policy-admin` and `payments-gateway`, as well as REST requests orchestrated by the [[entities/plans-service|Plans Service]].

```text
[ policy.policy.issued ]
          │
          ▼
   Plan Creation ─────────► Fetch Policy (policy-admin)
          │                 Fetch Customer Profile (customer-identity)
          │
          ▼
 Schedule Generation ─────► Calculate Fee (fees.InstalmentFee)
                            Generate Instalments (due dates & fee parts)
                            Persist Plan & Schedule (Store.Save)
          │
          ▼
   Active Servicing  ─────► Publish billing.instalment.due (5 days prior)
                            Collect via payments-gateway
                            Handle payments.collection.succeeded
          │
[ policy.policy.cancelled ]
          │
          ▼
    Plan Closure ─────────► Prorate(AnnualPremiumPence, StartDate, cancelledOn)
                            Persist Closure & Refund (Store.Close)
```

---

## 1. Plan Creation and Initiation

A plan's lifecycle begins when an insurance policy is issued in `policy-admin`.

1. **Trigger:** `billing-service` consumes the `policy.policy.issued` Pub/Sub event via subscription `billing-policy-issued` (or receives a `POST /v1/plans` REST request; see [[summaries/api-spec]]).
2. **Policy Ingestion:** `Service.CreateFromPolicy` calls `PolicyClient.Get` (`GET /v1/policies/{id}`) to fetch policy data including `ID`, `CustomerID`, `AnnualPremiumPence`, and `StartDate` (see [[entities/downstream-clients]]).
3. **Customer Data Snapshot:** Customer contact information is retrieved once at plan creation via `IdentityClient.Get` (`GET /v1/customers/{id}`) for correspondence. Note that `billing-service` intentionally does not subscribe to customer profile updates ([[ap:kb-tw-customer-platform-customer-identity/summaries/api-spec#customer-profile-updated|customer-identity (customer.profile.updated)]]), freezing the address for fee and arrears letters to the point of issuance.
4. **Plan Instantiation:** A `model.Plan` is created with:
   - `Status`: `"active"`
   - `Frequency`: `Monthly` or `Yearly`
   - `Instalments`: `12` for monthly plans, `1` for yearly plans
   - `FeePence`: Calculated by `fees.InstalmentFee(plan)` (see [[concepts/instalment-fee-calculation]] and [[decisions/0002-instalment-fee]])

See [[decisions/0001-billing-owns-plans]] for the architectural rationale behind billing owning plan lifecycles independently of policy administration.

---

## 2. Schedule Generation

Once the plan entity is formed, `schedule(plan)` constructs the collection schedule:

1. **Fee Distribution:** `fees.PerInstalment(plan)` calculates the fee allocated to each instalment (distributing any fractional penny remainders into earlier instalments).
2. **Base Amount Splitting:** The base annual premium is divided equally across instalments:
   $$\text{each} = \lfloor \text{AnnualPremiumPence} / \text{Instalments} \rfloor$$
3. **Instalment Construction:** For each instalment $i$ ($0 \le i < \text{Instalments}$):
   - `PlanID`: Assigned to the parent plan ID
   - `Number`: $i + 1$
   - `DueDate`: Offset by month from policy start date: `p.StartDate.AddDate(0, i, 0)`
   - `FeePence`: `feeParts[i]`
   - `AmountPence`: `each + feeParts[i]`
   - `Status`: `"due"`
4. **Storage:** The plan and all generated instalments are saved atomically via `Store.Save` (see [[entities/store]]).

---

## 3. Active Plan Servicing

During the active life of a plan, instalments transition through collection workflows:

- **Advance Notification:** 5 days prior to an instalment's `DueDate`, `billing.instalment.due` is published to notify `payments-gateway` and `notifications-hub` (see [[summaries/events]]).
- **Collection Success:** When a payment is collected, `payments-gateway` publishes [[ap:kb-tw-billing-payments-gateway/summaries/api-spec#payments-collection-succeeded|payments.collection.succeeded]], triggering `Store.MarkCollected` to update the instalment status.
- **Collection Retries and Arrears:** If collections fail repeatedly, `billing.payment.missed` is published. For detailed failure handling, see [[concepts/arrears-and-collections]].

---

## 4. Plan Cancellation and Closure

When a policy is cancelled prior to its natural expiration:

1. **Trigger:** `policy-admin` emits `policy.policy.cancelled`, received via subscription `billing-policy-cancelled`.
2. **Lookup:** `Service.Close` retrieves the existing plan record via `Store.ByPolicy`.
3. **Refund Calculation:** `Prorate(plan.AnnualPremiumPence, plan.StartDate, cancelledOn)` calculates the daily unearned premium refund due to the customer (see [[concepts/proration-and-refunds]]).
4. **State Finalization:** `Store.Close` updates the plan record in the database with the calculated refund amount and terminates active billing workflows.
