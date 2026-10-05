# Interfaces and references

API, event and module references for the application.

## Pages

- [REST API Specification](/summaries/api-spec.md) — The billing-service exposes an HTTP REST API defined in the OpenAPI 3.0.3 specification (billing-openapi.yaml) and mounted using the Chi router in internal/api/plans.go.
- [Pub/Sub Message Schemas and Events](/summaries/events.md) — billing-service relies on Google Cloud Pub/Sub for asynchronous event-driven workflows across the policy lifecycle, collection orchestration, and failure notification.
