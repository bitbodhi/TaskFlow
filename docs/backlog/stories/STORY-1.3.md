# STORY-1.3: Change a member's role (Owner only)

**Epic:** EPIC-1

As a board Owner, I want to change any member's role between Owner, Editor, and Viewer, so that I can adjust team permissions as the project evolves — without accidentally leaving a board with no Owner.

## Acceptance Criteria

**Given** I am a board Owner and the target member exists on this board **When** I PATCH `/boards/{boardId}/members/{userId}` with `{ role: "Editor" }` (or any valid role) **Then** the member's role is updated, I receive `200 OK` with the updated membership DTO (userId, displayName, email, role, addedAt), and subsequent reads reflect the new role.

**Given** I am a board Owner **When** I attempt to demote the last remaining Owner on the board (including demoting myself) **Then** I receive `409 Conflict` RFC 7807 ("Cannot remove the last Owner from a board") and the role is not changed.

**Given** I am a board Owner **When** I PATCH with an invalid `role` value or an empty body **Then** I receive `400` RFC 7807 validation problem and no change is made.

**Given** I am a board Owner **When** I PATCH the role of a `userId` who is not a member of this board **Then** I receive `404` RFC 7807 (member not found on this board) and no change is made.

**Given** I am an Editor or Viewer **When** I PATCH any member's role **Then** I receive `403` RFC 7807 (Owner-only action).

**Given** I am unauthenticated **When** I PATCH a member's role **Then** I receive `401` RFC 7807.

**Given** I am authenticated but not a member of the board **When** I PATCH a member's role **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** a role change is submitted via the UI **When** it is in idle, submitting, or error states **Then** the role selector is disabled while submitting, a success confirmation is displayed on completion, and the last-Owner conflict is surfaced as a user-readable error (not a generic 5xx).

**Given** the role selector is rendered **When** used by keyboard or assistive technology **Then** it meets WCAG 2.2 AA (labelled, focus-visible, role change announced via `role="status"`).

## NFR / constraints

- **Security / authorization:** Owner-only mutation; `403` for Editor/Viewer; `404` for non-members of the calling user; `401` unauthenticated. The last-Owner guard is enforced in the **Domain layer** (invariant on `Board` aggregate) so it cannot be bypassed at any call site.
- **Last-Owner protection:** the check counts active Owner memberships **before** applying the change; the domain raises a domain exception that the Application layer maps to `409`.
- **API:** DTOs only; RFC 7807 on every error path.
- **Clean Architecture:** `ChangeMemberRoleCommand` in Application; persistence in Infrastructure; invariant in Domain.
- **Real-time:** the role change is visible to other members <1s (via EPIC-6) — this story does not author SignalR code.
- **Accessibility:** WCAG 2.2 AA on role selectors; change confirmation via `role="status"`.

## UX spec (skeleton)

Within each member row in the members panel: an inline **role selector** (dropdown/select) showing the current role. **States:** **idle** (selector enabled, showing current role), **submitting** (selector disabled, spinner next to row), **success** (brief `role="status"` confirmation, e.g. "Role updated"), **error** (inline `role="alert"` message — including the last-Owner conflict message as plain language). Destructive implication (demoting an Owner): if changing from Owner and only one Owner remains, the selector should still submit and let the server reject with `409`; no client-side pre-blocking (server is authoritative). Keyboard: Tab to selector, arrow keys to change value, Enter/Space to confirm.

## Out of scope

- Bulk role changes.
- A dedicated "transfer ownership" flow — an Owner can promote another user to Owner, then demote themselves (two sequential PATCH calls).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.6, §2.2
**Tests:** `ChangeMemberRoleHandlerTests` (Application unit — success, last-Owner demote → 409, invalid role → 400, target not a member → 404, caller is Editor → 403, caller is non-member → 404); `ChangeMemberRoleEndpointTests` (Api integration — 200 updated DTO, 400, 401, 403, 404, 409, role persisted readable via GET); `MemberRoleSelector.spec.ts` (Vue/Vitest — idle/submitting/success/error states, last-Owner error message, a11y)
