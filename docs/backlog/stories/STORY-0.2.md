# STORY-0.2: Log in and receive a session

**Epic:** EPIC-0

As a registered user, I want to log in with my email and password and receive a session token, so that I can make authenticated requests for the rest of my session.

## Acceptance Criteria

**Given** a registered user submitting correct credentials **When** I log in **Then** I receive `200 OK` with a signed JWT and a user DTO (id, email); the token is signed with the Key Vault–sourced key and carries my user id and an expiry.

**Given** an unknown email or a wrong password **When** I log in **Then** I receive an RFC 7807 `401` with a **generic** message that does not reveal which field was wrong (no user enumeration), and no token is issued.

**Given** missing or malformed fields **When** I log in **Then** I receive an RFC 7807 `400` validation problem naming the offending field(s), and no token is issued.

**Given** a request to a protected endpoint **When** I present a valid unexpired JWT **Then** it is accepted; **When** the token is expired or tampered **Then** I receive `401` and the request is rejected.

**Given** I am on the SPA with an active session **When** any authenticated request returns `401` (e.g. the token expired while I was idle) **Then** the stored token is cleared and I am redirected to the login page — no silent failure or infinite retry.

**Given** the API is starting up **When** the Key Vault–held signing key cannot be loaded (managed identity not provisioned or misconfigured) **Then** the application **fails to start** with a structured error log entry — it never falls back silently to a dev/in-config key.

**Given** the login form **When** it is loading, submitting, or showing an auth error **Then** distinct loading and error states are shown, and the form meets WCAG 2.2 AA (labels, error association, keyboard operability, focus).

## NFR / constraints
- **Security:** JWT signing key sourced from **Key Vault via managed identity** (§2.10), never in config, with **fail-fast startup** if the key is unavailable (no silent dev-key fallback); token transmitted over HTTPS; **no user enumeration** on failed login; credentials never written to logs; **RFC 7807** on every error path; **DTOs only** at the API boundary.
- **Token handling (client):** the SPA stores the JWT in `sessionStorage` (cleared on tab close) and attaches it as `Authorization: Bearer <token>` on every subsequent API request — not an `HttpOnly` cookie (this is the agreed skeleton convention; the auth middleware and Vue HTTP client are built to match).
- **Clean Architecture (§2.2):** authenticate use-case in Application; token service + password verification in Infrastructure; thin controller + auth middleware in Api; `User` entity in Domain.
- **Reliability:** structured logging on the auth path (outcome only, no secrets); global error handler emits the 7807 envelope.
- **Accessibility:** WCAG 2.2 AA on the login form.

## UX spec (skeleton)
Stacked form: **email** + **password** fields, submit button, and a "need an account? Register" link. States: idle / submitting (disabled + spinner) / auth-error banner (`role="alert"`, generic "email or password is incorrect") / field-validation error (inline, `aria-describedby`). Keyboard order email → password → submit; focus the error banner on a failed login. No external comp — this note is the design reference.

## Out of scope
- Refresh tokens, "remember me", server-side logout/token revocation, MFA, password reset, social/OAuth login, and lockout/rate-limiting (all deferred).

**Design:** inline UX spec above (skeleton form — no external comp)
**Traces:** §1.3, §1.6, §2.2, §2.10
**Tests:** `AuthenticateUserHandlerTests` (Application unit — success issues token, bad-credentials, invalid input); `LoginEndpointTests` (Api integration — 200 token+DTO shape, 401 generic RFC 7807, 400 field errors); `JwtAuthMiddlewareTests` (Api integration — valid accepted, expired/tampered → 401); `StartupSigningKeyTests` (Api integration — app fails to start when the Key Vault key is unavailable, no dev-key fallback); `LoginForm.spec.ts` (Vue/Vitest — validation, loading/error states, a11y); `httpClient.spec.ts` (Vue/Vitest — a 401 response clears the stored token and redirects to login)
