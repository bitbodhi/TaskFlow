# STORY-1.2: Add a member to a board by email (Owner only)

**Epic:** EPIC-1

As a board Owner, I want to add a registered user to my board by their email address and assign them an initial role, so that they can immediately collaborate within the permissions I choose.

## Acceptance Criteria

**Given** I am a board Owner **When** I POST `{ email, role }` to `/boards/{boardId}/members` where `email` matches a registered user and `role` is one of `Owner`, `Editor`, or `Viewer` **Then** the user is added as a member with that role, I receive `201 Created` with the new membership DTO (userId, displayName, email, role, addedAt), and the board's member list reflects the addition.

**Given** I am a board Owner **When** I POST an `email` that is not registered in the system **Then** I receive `422 Unprocessable Entity` RFC 7807 with a human-readable detail ("No account found for that email address") and no membership is created.

**Given** I am a board Owner **When** I POST an `email` for a user who is already a member of the board **Then** I receive `409 Conflict` RFC 7807 ("User is already a member of this board") and no duplicate membership is created.

**Given** I am a board Owner **When** I POST an invalid `role` value or omit `email` **Then** I receive `400` RFC 7807 validation problem and no membership is created.

**Given** I am an Editor or Viewer **When** I POST to `/boards/{boardId}/members` **Then** I receive `403` RFC 7807 (Owner-only action).

**Given** I am unauthenticated **When** I POST to `/boards/{boardId}/members` **Then** I receive `401` RFC 7807.

**Given** I am authenticated but not a member of the board **When** I POST to `/boards/{boardId}/members` **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** a user is successfully added **When** the form is submitted **Then** the member immediately appears in the members list, a success confirmation is displayed, and the add-member form resets.

**Given** the add-member form is visible **When** it is in idle, submitting, or error states **Then** all states are visually distinct, the submit button is disabled while submitting, error messages are associated with their fields via `aria-describedby`, and the form meets WCAG 2.2 AA.

## NFR / constraints

- **Security / authorization:** Owner-only mutation; `403` for Editor/Viewer; `404` for non-members; `401` for unauthenticated callers. The email lookup MUST NOT disclose whether an email is registered when the caller is not an Owner (guard is: request rejected before lookup for non-Owners).
- **API:** DTOs only; RFC 7807 on every error path; `422` for unregistered email; `409` for duplicate.
- **Data:** `role` is validated against the `BoardRole` enumeration in the Domain; no free-text roles accepted.
- **Clean Architecture:** `AddBoardMemberCommand` in Application; user lookup + membership persistence in Infrastructure; no EF/Microsoft.* types in Domain.
- **Real-time:** the new member's appearance in other Owners' members panels propagates <1s (via EPIC-6) — this story does not author SignalR code.
- **Accessibility:** WCAG 2.2 AA on the add-member form; confirm or success feedback via `role="status"`.

## UX spec (skeleton)

Add-member control within the members panel: an **email** text input + **role** selector (`Owner` / `Editor` / `Viewer`, default `Editor`) + submit button ("Add member"). States: **idle** (form enabled), **submitting** (button disabled, spinner), **success** (brief `role="status"` toast or inline message, form resets), **validation error** (inline field-level message, `aria-describedby`), **server error / 422 / 409** (`role="alert"` banner with specific message). Email input: `type="email"`, `autocomplete="email"`. Role selector: `<select>` or accessible listbox. Keyboard: Tab through fields, Enter submits, Esc closes panel.

## Out of scope

- Email invitation with a pending-accept flow — this story requires the invitee to already be a registered user.
- Bulk add / CSV import.
- Notification email to the new member (future).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.6, §2.2
**Tests:** `AddBoardMemberHandlerTests` (Application unit — success Owner/Editor/Viewer role, unregistered email → 422, duplicate → 409, invalid role → 400, caller is Editor → 403, caller is non-member → 404); `AddBoardMemberEndpointTests` (Api integration — 201 DTO, 400, 401, 403, 404, 409, 422, membership persisted and readable via GET); `AddMemberForm.spec.ts` (Vue/Vitest — idle/submitting/success/error states, field validation, role selector, a11y)
