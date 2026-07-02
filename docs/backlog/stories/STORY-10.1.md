# STORY-10.1: Liveness & readiness health probes

**Epic:** EPIC-10

As an on-call engineer (or load-balancer orchestrator), I want the API to expose a DB-free liveness probe and a SQL-checking readiness probe, so that the platform can distinguish "process is up" from "process can serve traffic" without requiring authentication.

## Acceptance Criteria

**Given** the API process is running **When** anything (load balancer, monitoring tool, or operator) sends `GET /health` **Then** the response is `200 OK` with `Content-Type: application/json` body `{"status":"Healthy"}` — with no database call made, no authentication required, and the response time under 100 ms in any environment.

**Given** the API process is running and SQL Server is reachable **When** `GET /health/ready` is called **Then** the response is `200 OK` with body `{"status":"Healthy","checks":[{"name":"sqlserver","status":"Healthy"}]}` (or equivalent structured shape) and no authentication is required.

**Given** the API process is running but SQL Server is unreachable (network partition, SQL paused) **When** `GET /health/ready` is called **Then** the response is `503 Service Unavailable` with body `{"status":"Unhealthy","checks":[{"name":"sqlserver","status":"Unhealthy","description":"<non-sensitive error summary>"}]}` — no stack trace or connection-string value is included in the response body.

**Given** SQL recovers after a readiness failure **When** `GET /health/ready` is called again **Then** it returns `200 OK` (the check is live, not latched).

**Given** either probe is called **When** the response is returned **Then** it includes a `X-Correlation-Id` header (consistent with STORY-10.3) even though no authentication is required.

## NFR / constraints

- Both endpoints are **UNAUTHENTICATED** — no `[Authorize]` attribute, no JWT middleware applied to these routes.
- `/health` must be **DB-free** — no EF Core call, no SQL ping, no infrastructure dependency beyond process health.
- `/health/ready` SQL check uses ASP.NET Core's built-in `Microsoft.Extensions.Diagnostics.HealthChecks` with `AddSqlServer` (or equivalent EF Core check) — no custom polling loop.
- Health-check responses must **never** include connection strings, credentials, or internal stack traces.
- Both endpoints must respond in isolation (before middleware that would add auth, before SignalR negotiation, etc.); register them before `app.UseAuthentication()` in the pipeline ordering.
- Timeout for the SQL check: configurable, default 5 s; the probe returns `503` on timeout, not an unhandled exception.
- **Clean Architecture:** health-check registration lives in the `Infrastructure` project (SQL probe) and the `Api` project (endpoint mapping); no reference to `Microsoft.Extensions.Diagnostics.HealthChecks` from Domain or Application.
- **Pipeline order (shared constraint — identical in STORY-10.2):** the concrete middleware registration order in `Program.cs` MUST be: (1) exception-handling middleware (`UseExceptionHandler` / global error handler from STORY-10.2) registered FIRST as the outermost layer; (2) correlation-id middleware (STORY-10.3) registered second; (3) health endpoints (`MapHealthChecks`) mapped next — they resolve before authentication so they are always reachable; (4) `UseRouting`; (5) `UseAuthentication` / `UseAuthorization`. The health endpoints bypass auth because they are mapped before `UseAuthentication`. The error handler wraps everything including health checks.

## Out of scope

- Deep health checks for SignalR, Key Vault, or Sentry (those surface via their own SDK exceptions and Sentry captures).
- A UI health dashboard or status page exposed to end-users.
- Custom health-check serialization beyond ASP.NET Core's default `HealthStatus` JSON writer (unless `HealthCheckOptions.ResponseWriter` is needed to strip secrets — then only that).

**Traces:** §2.9, §1.6
**Tests:** `LivenessProbeTests` (Api integration — GET /health returns 200 with no DB call; assert no EF Core query logged); `ReadinessProbeTests` (Api integration — 200 when SQL reachable, 503 when SQL host unreachable/fake connection string; recovery: 200 after host restored; assert response body shape; assert no connection string in body); `MiddlewarePipelineOrderTests` (Api integration — assert health endpoints return 200 when `UseAuthentication` is not yet in the pipeline / no JWT provided; assert exception handler is outermost by verifying a fault before routing still returns RFC 7807); no Vitest tests (probe is backend-only).
