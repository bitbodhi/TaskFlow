# STORY-10.4: Sentry integration (backend + SPA + CI source maps)

**Epic:** EPIC-10

As an on-call engineer, I want unhandled errors from both the API and the Vue SPA captured automatically in Sentry — with de-minified traces (source maps uploaded by CI) and a Sentry alert wired as the `incident.opened` signal — so that production incidents are surfaced immediately and traceable to source.

## Acceptance Criteria

**Given** the API is configured with a Sentry DSN **When** an unhandled exception is caught by the global error handler (STORY-10.2) **Then** `Sentry.CaptureException` is called (via `Sentry.AspNetCore` automatic capture or explicit call), the event appears in the Sentry project, and the event's `tags` include the `CorrelationId` from STORY-10.3's scope.

**Given** the API starts without a Sentry DSN configured (env variable absent or empty) **When** the process starts **Then** the application starts normally, logs a `Warning`-level entry ("Sentry DSN not configured — error capture disabled"), and handles all requests as normal — Sentry being absent does not crash or degrade the API.

**Given** the backend Sentry DSN **When** it is read at startup **Then** it is sourced from the `SENTRY_DSN` app setting (or Key Vault reference via managed identity) — the value never appears in source code, `appsettings.json`, or any build artifact.

**Given** the Vue SPA is loaded in a browser **When** the `@sentry/vue` SDK is initialised **Then** it reads the Sentry DSN from the runtime `config.json` (fetched at boot, not baked into the bundle), initialises with `Vue.config.errorHandler` wired, and captures unhandled Vue errors and promise rejections.

**Given** the SPA DSN is public **When** the SPA bundle is inspected or `config.json` is fetched **Then** the DSN is an ingestion-scoped token (write-only; no read or admin scope) — confirming that leaking it does not expose event data.

**Given** the SPA encounters an unhandled error caught by the `@sentry/vue` error handler **When** the error event is sent to Sentry **Then** the Sentry event includes a de-minified stack trace because CI has uploaded the source map for that release.

**Given** a CI build succeeds and produces a release **When** the pipeline completes the deploy step **Then** the pipeline runs `sentry-cli releases new <version>` and `sentry-cli releases files <version> upload-sourcemaps ./dist` (or the equivalent Sentry Azure Pipelines task), associating the release with the deployed artifact version.

**Given** a Sentry alert is configured to fire when the first occurrence of any new issue appears in the production environment **When** that alert fires **Then** Sentry's alert rule delivers a WEBHOOK to a dedicated receiver endpoint hosted in the API (e.g. `POST /internal/sentry-webhook`); the receiver authenticates the request via a shared-secret header (`SENTRY_WEBHOOK_SECRET`); the receiver invokes `metrics-log.sh incident.opened` (or equivalent in-process call), producing an `incident.opened` event in the metrics ledger — connecting Sentry to the spine's `change-failure` producer. There is no "or pipeline script" alternative: the webhook→receiver→metrics-log.sh path is the sole mechanism.

**Given** the `sentry-cli` source-map upload step runs in CI **When** the upload fails (network error, invalid token, Sentry API unavailable) **Then** the deploy pipeline continues — the source-map upload is a SOFT gate; the pipeline logs a `warning`-level entry ("Sentry source-map upload failed — this release's traces will be minified until re-uploaded"); the deploy artifact is still promoted; a re-upload can be triggered manually once the issue is resolved. The deploy MUST NOT be blocked by a monitoring-tooling failure.

**Given** the SPA's `@sentry/vue` integration is active **When** the application transitions between routes **Then** Sentry records a breadcrumb for each navigation so that the event context shows the user's path before an error.

## NFR / constraints

- **Backend SDK:** `Sentry.AspNetCore` — registered via `UseSentry()` in `Program.cs`; automatic capture of unhandled exceptions; `SendDefaultPii = false` (no PII in Sentry payloads).
- **SPA SDK:** `@sentry/vue` — initialised in `main.ts` after `app = createApp(App)`; DSN read from `window.__config.sentryDsn` (populated from runtime `config.json`); graceful no-op when DSN is absent/empty.
- **DSN handling:** backend DSN = app setting / Key Vault reference (managed identity); SPA DSN = ingestion-scoped, public, in `config.json`. Neither DSN value is in source code or committed configuration files.
- **Release tracking:** CI pipeline sets `SENTRY_RELEASE` to the build number or commit SHA; both `sentry-cli` upload steps reference the same value so front-end and back-end events are grouped to one release in Sentry.
- **Source maps:** Vite build produces source maps; CI uploads them to Sentry **after** building and **before** deploying; the `dist/` source maps are **not** served publicly (deleted or excluded from the frontend App Service deploy artifact after upload).
- **`SendDefaultPii = false`:** confirmed in both SDK initialisations; user email/IP are not captured in Sentry events unless explicitly opt-in in a future story.
- **Clean Architecture:** Sentry SDK is referenced only from the `Api` project (backend) and `src/main.ts` (SPA); no Sentry import in Domain, Application, or SPA component files.
- **`incident.opened` mechanism:** Sentry alert rule → WEBHOOK → API receiver endpoint (`POST /internal/sentry-webhook`) authenticated with `SENTRY_WEBHOOK_SECRET` → receiver calls `metrics-log.sh incident.opened`. This is the only path; no CI-script alternative exists. The secret is stored as an app setting / Key Vault reference and never committed to source.
- **Source-map upload soft gate:** `sentry-cli` upload failure is non-fatal; the pipeline emits a warning and proceeds. The CI step must use `continue-on-error: true` (or equivalent) so an upload failure never blocks a deploy.
- The `incident.opened` alert binding is a **one-alert, one-signal** binding; complex alert routing is out of scope.

## Out of scope

- Sentry performance monitoring / tracing (transactions, spans) — error capture only in v1.0.
- User-feedback prompts on crash (Sentry's user-feedback widget).
- Multiple Sentry environments beyond `production` and `development` (staging is additive).
- Alert routing beyond the single `incident.opened` signal (PagerDuty, Slack, etc.).

**Traces:** §1.6, §2.10, ADR-000 R3
**Tests:** `SentryIntegrationTests` (Api integration — exception captured via `ISentryClient` mock or `SentryOptions.BeforeSend` hook; DSN-absent path: app starts, `Warning` logged, no exception thrown; assert `SendDefaultPii = false`; webhook receiver: valid secret + Sentry payload → `incident.opened` written to ledger; invalid/missing secret → 401 and no ledger write; source-map upload failure path: pipeline warning logged, exit code 0 / deploy proceeds); `SentryInit.spec.ts` (Vitest — SDK initialised with DSN from config mock; absent DSN → no-op init; `Vue.config.errorHandler` is set; breadcrumb recorded on route change).
