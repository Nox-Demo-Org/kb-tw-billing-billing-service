# Architecture decisions

One ADR per architecture decision the code or documents make evident.

## Pages

- [ADR-0001: Billing Owns Payment Plans](/decisions/0001-billing-owns-plans.md) — Under this architecture: - policy-admin remains the source of truth for the policy and its annual premium.
- [ADR-0002: Instalment fee on monthly plans](/decisions/0002-instalment-fee.md) — Key implementation rules established in internal/fees/instalment.go: - The fee applies only to plans with frequency model.Monthly; yearly plans are charged £0.
