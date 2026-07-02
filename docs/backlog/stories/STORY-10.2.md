# STORY-10.2: Global error handler → RFC 7807 responses

**Epic:** EPIC-10

As an on-call engineer, I want every unhandled exception from the API to produce a structured RFC 7807 problem-details response with a correlation id — and never a raw stack trace or internal detail — so that clients receive consistent, safe error shapes and I can trace any incident back to a specific request.

## Acceptance Criteria

**Given** any request to the API triggers an unhandled exception (null reference, unexpected domain error, infrastructure failure) **When** the global error handler catches it **Then** the response is an RFC 7807 problem-details body (`application/problem+json`) with at minimum `type`, `title`, `status`, and `traceId` fields; the `status` is `500`; no stack trace, no exception message, no internal type name, and no PII is included in the response body.

**Given** the error handler produces a `500` response **When** the response is serialized **Then** the `traceId` field value matches the `X-Correlation-Id` response header on that same request (set by STORY-10.3's middleware, or a fallback `HttpContext.TraceIdentifier` if 10.3 is not yet merged).

**Given** an unhandled exception is caught **When** the handler logs it **Then** it logs at `Error` level with the full exception (message + stack) in the structured log sink, keyed by the same correlation id — the stack trace is in the logs, never in the response.

**Given** an `OperationCanceledException` or `TaskCanceledException` is thrown because the client disconnected **When** the error handler processes it **Then** it returns `499` (or swallows cleanly without a `500`) and logs at `Information` or `Warning` level — not `Error`.

**Given** any error response is produced **When** a client parses it **Then** the `Content-Type` is `application/problem+json` and the body is valid JSON conforming to RFC 7807 (the shape all other epics' error ACs depend on).

**Given** the API is running normally with no unhandled exceptions **When** requests succeed **Then** the error handler adds no observable overhead (it is middleware, not a try/catch on every endpoint).

## NFR / constraints

- Implemented as ASP.NET Core middleware (or `IExceptionHandler` / `app.UseExceptionHandler`) — a **single global registration**, not per-controller try/catch blocks.
- The handler must be registered **before** `UseRouting`/`UseAuthentication` in the pipeline so it catches errors from all middleware layers.
- **No secrets, PII, or internal stack traces** in any response body — ever. The check is mechanical: the serialized response body must not contain the string `StackTrace`, `at System.`, or the exception's `.Message` property for unhandled exceptions.
- `traceId` in the response body and `X-Correlation-Id` in the response header use the **same value** (correlation id from STORY-10.3 or `HttpContext.TraceIdentifier` fallback).
- Validation errors (`400`) produced by model binding and `FluentValidation`/`DataAnnotations` are handled by ASP.NET Core's `problem()` factory (already RFC 7807-conformant) — the global handler does not need to re-serialize these; it only catches **unhandled** exceptions (status `500` path).
- **Clean Architecture:** the error-handler middleware lives in the `Api` project; it must not import Domain or Application types to produce the response.
- A developer-mode detail extension (`detail` field showing the exception message) is permissible **only** when `IHostEnvironment.IsDevelopment()` is true and must be provably absent in staging/production.
- **Pipeline order (shared constraint — identical in STORY-10.1):** the concrete middleware registration order in `Program.cs` MUST be: (1) exception-handling middleware (`UseExceptionHandler` / global error handler from this story) registered FIRST as the outermost layer; (2) correlation-id middleware (STORY-10.3) registered second; (3) health endpoints (`MapHealthChecks`) mapped next — they resolve before authentication so they are always reachable; (4) `UseRouting`; (5) `UseAuthentication` / `UseAuthorization`. The health endpoints bypass auth because they are mapped before `UseAuthentication`. The error handler wraps everything including health checks.

## Out of scope

- Per-exception mapping to specific `4xx` codes (e.g. mapping `NotFoundException` → `404`) — that is domain-error translation handled per-use-case in later epics.
- Sentry capture of errors (STORY-10.4 layers that on top of the structured log entry produced here).
- Retry / circuit-breaker logic (resilience patterns are out of scope for v1.0).

**Traces:** §1.6, §2.2
**Tests:** `GlobalErrorHandlerTests` (Api integration — unhandled exception → 500 RFC 7807 body with `traceId`; assert `Content-Type: application/problem+json`; assert body contains no `StackTrace` substring; assert `X-Correlation-Id` header equals body `traceId`; cancelled request → not 500; development mode includes `detail`; production mode omits `detail`); `MiddlewarePipelineOrderTests` (Api integration — assert exception handler catches errors thrown before routing; assert health endpoints return 200 with no auth token present; assert pipeline order matches the shared constraint: exception handler outermost, then correlation-id, then health, then routing, then auth).
