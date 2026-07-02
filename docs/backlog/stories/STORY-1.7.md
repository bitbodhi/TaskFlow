# STORY-1.7: Current-user endpoint and sign-out

**Epic:** EPIC-1

As an authenticated user, I want to retrieve my own profile from the server and sign out of the application, so that the UI can display my identity accurately and I can end my session cleanly on any device.

## Acceptance Criteria

**Given** I am authenticated **When** I call `GET /me` **Then** I receive `200 OK` with a user DTO containing my id, display name, and email — derived from the JWT claims without a database write.

**Given** I am unauthenticated (missing, expired, or invalid token) **When** I call `GET /me` **Then** I receive `401` RFC 7807.

**Given** I am authenticated and the app header is rendered **When** the page loads **Then** my display name (and/or avatar initials) is shown in the header, populated from the `GET /me` response, before any board-level data loads.

**Given** I click "Sign out" in the app **When** I confirm (or the action has no confirm prompt — sign-out is not destructive) **Then** my JWT is removed from client storage (localStorage / sessionStorage / cookie), the Pinia auth store is cleared, I am navigated to the login page, and no subsequent API request carries the old token.

**Given** I sign out **When** I press the browser back button **Then** protected routes redirect me to the login page (client-side route guard) and no board data is displayed.

**Given** the sign-out control is rendered **When** used by keyboard or assistive technology **Then** it is reachable by Tab, activatable by Enter/Space, and the resulting navigation to the login page is announced appropriately. WCAG 2.2 AA.

**Given** `GET /me` fails (network error or 5xx) **When** the app header renders **Then** an error state is shown (e.g. "Could not load user") and sign-out is still accessible (does not depend on a successful `/me` response).

## NFR / constraints

- **Security / authorization:** `GET /me` is read-only and serves only the authenticated caller's own data (derived from JWT `sub` / `email` / `name` claims — no DB write). `401` for any unauthenticated call.
- **Stateless JWT:** sign-out is **client-side only** — no token blacklist, no server-side session invalidation. The JWT remains cryptographically valid until its `exp` claim; this is an accepted trade-off for stateless auth (documented in Out of scope). The client MUST clear the token from all storage locations and clear auth state from Pinia.
- **No database write on `GET /me`:** the endpoint reads claims from the validated JWT; it does not update `LastLoginAt` or any other field (that is a future concern).
- **API:** RFC 7807 on every error path; DTO only at the boundary.
- **Clean Architecture:** `GetCurrentUserQuery` in Application (reads claims from `ICurrentUserService`); thin endpoint in Api; no EF query needed.
- **Client route guard:** all board/list/card routes are protected; unauthenticated navigation redirects to `/login` with the original path preserved as a redirect query param.
- **Accessibility:** WCAG 2.2 AA on sign-out control and user identity display in header.

## UX spec (skeleton)

**App header:** right-hand side shows display name and/or avatar (initials fallback). Clicking opens a small **user menu** with "Sign out" option. **States:** **loading** (placeholder/skeleton in header), **populated** (display name + avatar), **error** ("Could not load user" with sign-out still accessible). **Sign-out:** no confirm dialog (signing out is reversible by signing back in — not destructive). On click: immediately clear token + auth store, navigate to `/login`. Keyboard: user menu button is focusable (Tab), menu opens on Enter/Space, menu items navigable by arrow keys, Esc closes menu and returns focus to trigger.

**Login redirect:** when a route guard fires on an unauthenticated navigation, redirect to `/login?redirect=<original-path>`. On successful login, navigate to the saved path.

## Out of scope

- Server-side token revocation / blacklist (stateless JWT trade-off — acceptable for v1, noted as future concern).
- "Sign out all devices" / session list (future).
- Profile editing (display name, email, password change) — future.
- Social/OAuth login — deferred (EPIC-0 notes).

**Traces:** §1.3, §1.6, §2.2
**Tests:** `GetCurrentUserHandlerTests` (Application unit — success returns DTO from claims, unauthenticated → 401); `GetCurrentUserEndpointTests` (Api integration — 200 DTO with valid token, 401 with no/invalid token); `AuthStore.spec.ts` (Vue/Vitest — sign-out clears token + Pinia state, route guard redirects unauthenticated navigation, redirect param preserved and used after login); `AppHeader.spec.ts` (Vue/Vitest — loading/populated/error states, sign-out accessible by keyboard, a11y)
