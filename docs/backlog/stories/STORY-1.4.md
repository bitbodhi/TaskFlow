# STORY-1.4: Remove a member from a board (Owner only)

**Epic:** EPIC-1

As a board Owner, I want to remove a member from a board after confirming the action, so that I can revoke access when someone leaves the project — without risking accidental removal of the last Owner.

## Acceptance Criteria

**Given** I am a board Owner and the target member is not the last Owner **When** I confirm removal and DELETE `/boards/{boardId}/members/{userId}` **Then** the membership is deleted, I receive `204 No Content`, and the member no longer appears in `GET /boards/{boardId}/members`.

**Given** I am a board Owner **When** I attempt to remove the last remaining Owner on the board (including removing myself when I am the only Owner) **Then** I receive `409 Conflict` RFC 7807 ("Cannot remove the last Owner from a board") and the membership is not deleted.

**Given** I am a board Owner **When** I DELETE a `userId` who is not a member of this board **Then** I receive `404` RFC 7807 (member not found on this board) and nothing changes.

**Given** I am an Editor or Viewer **When** I DELETE any member **Then** I receive `403` RFC 7807 (Owner-only action).

**Given** I am unauthenticated **When** I DELETE a member **Then** I receive `401` RFC 7807.

**Given** I am authenticated but not a member of the board **When** I DELETE a member **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** I initiate removal via the UI **When** I click the remove control **Then** a confirmation dialog appears naming the member ("Remove Jane Doe from this board?") with a destructive-confirm button and a cancel button before any request is sent.

**Given** I confirm removal **When** the request is in flight **Then** the confirm button is disabled/replaced with a spinner, preventing double-submission.

**Given** removal succeeds **When** the response is `204` **Then** the member row is removed from the list without a full page reload, and a `role="status"` announcement confirms the action.

**Given** removal fails (409 last-Owner, or any 5xx) **When** the dialog is still open **Then** an error message is displayed within the dialog (`role="alert"`), the dialog remains open, and the member is not removed from the visible list.

**Given** the confirm dialog and remove controls are rendered **When** used by keyboard or assistive technology **Then** focus is trapped within the dialog when open, focus returns to the triggering element on close, and the dialog meets WCAG 2.2 AA.

## NFR / constraints

- **Security / authorization:** Owner-only mutation; `403` for Editor/Viewer; `404` for non-members (calling user); `401` unauthenticated. Last-Owner guard enforced in the Domain layer (same invariant as STORY-1.3).
- **Destructive action:** a confirm step is mandatory before the DELETE request is sent (per §1.6 usability requirement); no undo.
- **Self-removal:** an Owner removing themselves is permitted if at least one other Owner exists. The Domain invariant is the sole guard — no special client-side logic for self vs. other.
- **API:** DTOs only; RFC 7807 on every error path; `204` on success (no body).
- **Clean Architecture:** `RemoveBoardMemberCommand` in Application; persistence in Infrastructure; invariant in Domain.
- **Real-time:** the removal is reflected for other members <1s (via EPIC-6) — this story does not author SignalR code.
- **Accessibility:** WCAG 2.2 AA; confirm dialog focus trap; destructive button styled distinctively (but not by colour alone).

## UX spec (skeleton)

Each member row in the members panel has a **remove** icon/button (visible on hover or always-visible for Owners). Activating it opens a **confirm dialog** (modal): title "Remove member", body "Remove [Display Name] from this board? They will lose all access immediately.", two buttons — "Remove" (destructive, e.g. red-tinted but also has distinctive label) and "Cancel". **Dialog states:** **idle** (both buttons enabled), **submitting** (Remove button disabled/spinner, Cancel disabled), **error** (`role="alert"` inline message, both buttons re-enabled). On success: dialog closes, member row fades/removes from list, `role="status"` announcement. Keyboard: Esc cancels dialog; Tab cycles within dialog (focus trap); focus returns to triggering row (or next row) on close.

## Out of scope

- Revoking access tokens in-flight (stateless JWT — the removed member's existing tokens are valid until expiry; token revocation is a future concern).
- Notifying the removed member (future).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.6, §2.2
**Tests:** `RemoveBoardMemberHandlerTests` (Application unit — success, last-Owner remove → 409, target not a member → 404, caller is Editor → 403, caller is non-member → 404, self-removal with other Owner present → success, self-removal as sole Owner → 409); `RemoveBoardMemberEndpointTests` (Api integration — 204, 401, 403, 404, 409, membership absent in GET after removal); `RemoveMemberDialog.spec.ts` (Vue/Vitest — confirm dialog open/cancel/confirm, submitting state, error state, 409 last-Owner message, focus trap, a11y)
