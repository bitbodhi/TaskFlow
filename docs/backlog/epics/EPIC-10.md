# EPIC-10: Observability, Health & Resilience

**Goal:** The running system is observable, fails safe, and is probe-ready for load balancers and orchestrators. Unhandled errors are captured centrally in Sentry and surface to callers as consistent RFC 7807 problem responses. Engineers and on-call responders can correlate every request, confirm process health, and trust that no secrets or PII are ever leaked through logs or error responses.
**FRs covered:** §1.6 (reliability, structured logging, global error handling); §2.9 (health probes `/health` liveness + `/health/ready` readiness); §2.10 (secrets handling — Sentry DSN via app setting/Key Vault for the backend, public ingestion-scoped DSN in runtime `config.json` for the SPA); ADR-000 R3 (Sentry replaces App Insights; CI uploads release + source maps; Sentry alert wires `incident.opened`).
**Traces:** §1.6, §2.9, §2.10, ADR-000 R3

## Stories

- STORY-10.1 — Liveness & readiness health probes
- STORY-10.2 — Global error handler → RFC 7807 responses
- STORY-10.3 — Structured logging with correlation id
- STORY-10.4 — Sentry integration (backend + SPA + CI source maps)
- STORY-10.5 — SPA reliability audit: loading/empty/error states + WCAG 2.2 AA sweep

## Out of scope

- Application-level alerting beyond the single Sentry alert that wires `incident.opened` (on-call routing, PagerDuty, etc.).
- Distributed tracing across multiple services (out of scope for v1.0 single-API topology).
- Per-board activity feed and user-facing audit history — that is EPIC-7 (product history for members), not this epic (operability for engineers/on-call).
- Custom dashboards, SLO burn-rate alerts, or uptime monitors beyond what Sentry provides out-of-the-box.
- JWT-key fail-fast on startup — that is EPIC-0 STORY-0.2's harder guarantee and is not re-introduced here.

## Notes

- EPIC-7 is the user-facing per-board activity feed. EPIC-10 is exclusively the engineering/ops layer: probes, error handling, logging, and Sentry. Do not conflate them.
- Stories are ordered by dependency: 10.1 (probes) and 10.2 (error handler) are infrastructure prerequisites; 10.3 (logging) enriches error responses from 10.2; 10.4 (Sentry) layers over the error handler and logger; 10.5 (SPA audit) is a cross-cutting sweep that can proceed in parallel once 10.2 is merged.
- All stories share the same NFR baseline: no secrets/PII in logs, correlation id on every response, RFC 7807 on every error path, UNAUTHENTICATED probe endpoints.
