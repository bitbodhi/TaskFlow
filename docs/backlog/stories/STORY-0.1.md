# STORY-0.1: Register an account

**Epic:** EPIC-0

As a new visitor, I want to register with an email and password, so that I have an account that owns my boards and cards.

## Acceptance Criteria

**Given** I am an unregistered visitor on the registration page **When** I submit a well-formed, unused email and a password that meets the policy **Then** my account is created and persisted in SQL, the password is stored only as a salted hash, and I receive a `201 Created` with a DTO containing my id and email (never the password/hash).

**Given** an email that is already registered **When** I submit registration with that email **Then** I receive an RFC 7807 problem response with status `409 Conflict`, and no second account is created.

**Given** two registration requests for the same new email arrive concurrently **When** both are processed **Then** exactly one account is created and the other returns the same `409 Conflict` RFC 7807 response — enforced by a **unique index on `Users.Email`**, not by an application-layer check alone.

**Given** invalid input (malformed email, or a password failing the policy) **When** I submit registration **Then** I receive an RFC 7807 `400` validation problem that names the offending field(s), and no account is created.

**Given** the registration form **When** it is loading, submitting, or showing a server error **Then** distinct loading and error states are shown, the form is fully keyboard-operable with programmatic labels and focus management, and it meets WCAG 2.2 AA.

**Given** a successfully created account **When** I inspect the stored row **Then** the password column holds a salted one-way hash only — plaintext never appears in the database or in logs.

## NFR / constraints
- **Security:** password stored as a salted hash (ASP.NET Core Identity / PBKDF2-class); input validated server-side; **DTOs only at the API boundary** (no entity leakage); **RFC 7807** for every error path; secrets (JWT signing key) sourced from Key Vault, not config (§2.10) — relevant once login is added in STORY-0.2.
- **Password policy (the "policy" the ACs refer to):** minimum **8** characters, maximum **128**; **no** mandatory character-class complexity (per NIST SP 800-63B); enforced via ASP.NET Core Identity `PasswordOptions` and surfaced as the RFC 7807 `400` when violated.
- **Clean Architecture (§2.2):** `User` entity in Domain; register use-case in Application; EF Core persistence + password hasher in Infrastructure; thin controller in Api. Dependencies point inward only.
- **Reliability:** EF Core migration creates the `Users` table with a **unique index on `Email`** (concurrency-safe duplicate rejection); structured logging on the register path (no PII/secrets in logs); global error handler emits the 7807 envelope.
- **Accessibility:** WCAG 2.2 AA on the registration form (labels, error association, focus order, contrast).
- This story has **no server-side authorization** AC — registration is the pre-authentication boundary itself.

## UX spec (skeleton)
Standard stacked form: **email** + **password** (+ confirm) fields, submit button. States: idle / submitting (disabled button + spinner) / field-level validation error (inline, `aria-describedby`) / server-error banner (`role="alert"`). Keyboard order email → password → confirm → submit; on failed submit, focus moves to the first invalid field. No external visual comp for this skeleton form — this note is the design reference.

## Out of scope
- Login / JWT issuance (STORY-0.2), email verification, password reset, social/OAuth login, account lockout, rate-limiting, and CAPTCHA.
- Board/card creation (later stories in this epic).

**Design:** inline UX spec above (skeleton form — no external comp)
**Traces:** §1.3, §1.6, §2.2, §2.10
**Tests:** `RegisterUserHandlerTests` (Application unit — success, duplicate-email → conflict, invalid-input → validation); `RegisterEndpointTests` (Api integration — 201 DTO shape, 409 + RFC 7807 body, 400 + RFC 7807 field errors, hash-not-plaintext persistence assertion); `RegisterForm.spec.ts` (Vue/Vitest — client validation, loading/error states, a11y labels & keyboard operability)
