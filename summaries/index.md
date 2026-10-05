# Interfaces and references

API, event and module references for the application.

## Pages

- [REST API Specification](/summaries/api-spec.md) — The billing-service exposes an HTTP REST API defined in the OpenAPI 3.0.3 specification (billing-openapi.yaml) and mounted using the Chi router in internal/api/plans.go.
- [Events Specification](/summaries/events-spec.md) — billing-service integrates asynchronously with other Tidewell Mutual services using Google Cloud Pub/Sub topics.
