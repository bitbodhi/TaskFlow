# STORY-10.3: Structured logging with correlation id

**Epic:** EPIC-10

As an on-call engineer, I want every API request to carry a consistent correlation id through all log entries — with structured, levelled output and no secrets or PII — so that I can trace the full lifecycle of any request from a single search in the log sink.

## Acceptance Criteria

**Given** any inbound HTTP request **When** it passes through the correlation middleware **Then** a correlation id is assigned: if the request carries an `X-Correlation-Id` header with a non-empty value that header's value is used; otherwise a new `Guid` is generated. The id is attached to the `ILogger` scope for the duration of the request and echoed in an `X-Correlation-Id` response header.

**Given** a correlation id is active **When** any `ILogger` call is made inside that request's scope (controllers, use-case handlers, infrastructure services) **Then** the structured log entry includes a `CorrelationId` property matching the request's id — no additional per-call instrumentation required from feature code.

**Given** the application is configured for structured logging (Serilog or Microsoft.Extensions.Logging with a structured sink) **When** a log entry is emitted **Then** it is in JSON/structured format (not plain-text concatenation) with at minimum `Timestamp`, `Level`, `CorrelationId`, `Message`, and for request-scoped entries `RequestPath` and `StatusCode`.

**Given** log entries are written **When** any entry is inspected **Then** it contains no connection strings, JWT tokens, bearer values, Key Vault URIs, passwords, or PII (user email addresses, names). Structured properties from sensitive inputs are excluded or masked at the logging layer.

**Given** log levels are configured **When** the application runs in production **Then** `Information` and above are emitted; `Debug`/`Trace` are suppressed unless explicitly overridden by configuration. Level overrides are config-driven (appsettings.json / environment variable), not code-driven.

**Given** Sentry is integrated (STORY-10.4) **When** it is configured **Then** the structured log entries feed Sentry breadcrumbs automatically via the SDK's `AddSentry()` log sink integration — no additional instrumentation needed beyond registering the sink.

## NFR / constraints

- Correlation middleware is registered **before** `UseAuthentication` and `UseAuthorization` so the id is available on all requests including 401/403 responses.
- The middleware **reads and propagates** a caller-supplied `X-Correlation-Id`; it does not require one. Downstream services (SignalR hub, background jobs) pass the id through `ILogger` scope.
- **No secrets/PII in logs** is a hard constraint, not best-effort. Confirm by asserting in tests that the structured log sink never contains `"password"`, `"token"`, or `"connstr"` property keys.
- Logging configuration (sink, level, output format) lives in `appsettings.json` / `appsettings.{Environment}.json` — not hard-coded in `Program.cs` beyond bootstrapping.
- **Clean Architecture:** `ILogger<T>` is used throughout Application and Infrastructure (it is a Microsoft abstraction acceptable in Application per the framework conventions); no Serilog/Sentry types in Domain or Application.
- Log entries for health probe endpoints (`/health`, `/health/ready`) may be suppressed or throttled to avoid log noise from frequent load-balancer pings (configurable, default: suppress `Information` from health-check middleware).

## Out of scope

- Distributed tracing (`ActivitySource`, OpenTelemetry) — correlation id via header propagation is sufficient for v1.0.
- Log shipping infrastructure (Log Analytics workspace, Elastic, etc.) — the sink target is Sentry (STORY-10.4) and the local console in development; cloud log aggregation beyond Sentry is a future concern.
- Request/response body logging (risk of PII capture).

**Traces:** §1.6, §2.10
**Tests:** `CorrelationMiddlewareTests` (Api integration — inbound id propagated to response header and log scope; no inbound id → new Guid generated and echoed; assert `CorrelationId` property present in captured log entries; assert no sensitive property keys in log output); `LoggingConfigTests` (unit — `Debug` suppressed in production config; `Information` passes through; level overridable via config key).
