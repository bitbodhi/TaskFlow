# STORY-3.3: Delete a card

**Epic:** EPIC-3

As an Editor or Owner of a board, I want to delete a card after confirming the action, so that I can remove tasks that are no longer relevant without accidentally losing work.

## Acceptance Criteria

**Given** I am an authenticated Editor or Owner of the board **When** I DELETE `/cards/{cardId}` **Then** the card is permanently removed from the database, I receive `204 No Content`, and the card no longer appears in any list-cards response.

**Given** I am a Viewer on the board **When** I DELETE `/cards/{cardId}` **Then** I receive `403` RFC 7807 and the card remains in the database.

**Given** I am not a member of the board, or present no/invalid JWT **When** I DELETE `/cards/{cardId}` **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated) and the card is unchanged.

**Given** `cardId` does not exist **When** DELETE is requested **Then** I receive `404` RFC 7807 — indistinguishable from the non-member 404.

**Given** I am an Editor or Owner viewing a card **When** I trigger delete in the UI **Then** a confirmation dialog appears describing what will be deleted (card title) with explicit **Cancel** and **Delete** actions. Dismissing or pressing `Esc` cancels without any change.

**Given** I confirm deletion in the UI **When** the DELETE request succeeds **Then** the card is removed from the board/list view immediately (optimistic or server-confirmed removal — design TBD), and focus returns to the list column it belonged to.

**Given** the confirmation dialog or delete button renders **When** viewed by a Viewer **Then** the delete control is not shown or is disabled — Viewers cannot reach the confirmation step.

**Given** the confirmation dialog is displayed **When** rendered **Then** it meets WCAG 2.2 AA: `role="alertdialog"`, `aria-modal="true"`, `aria-labelledby`, focus moves into the dialog on open, focus is trapped within it, `Esc` cancels.

## NFR / constraints
- **Authorization:** Editor and Owner may delete; Viewer receives `403`. Enforced server-side on every request.
- **Destructive action:** confirmed in the UI before the DELETE is issued. The confirmation prompt states the card title so the user has enough context to decide.
- **Cascade:** deleting a card removes it and any future child entities (comments — EPIC-5; activity entries — EPIC-7) via EF cascade delete. This story defines the cascade contract; child tables do not yet exist in EPIC-3 but the delete endpoint must be forward-compatible (cascade configured on the migration, not ad-hoc).
- **Real-time:** other board members' UIs will not reflect the deletion in real time until EPIC-6 broadcasts the event — note as a constraint, not authored here.
- `204 No Content` on success (no body); RFC 7807 on all error paths.

## UX spec (skeleton)
Delete is a secondary action available in the card detail panel (e.g. a "…" menu or a "Delete card" button in a danger zone at the bottom). **Confirmation dialog:** heading "Delete card?", body "'{cardTitle}' will be permanently deleted. This cannot be undone.", two buttons: **Cancel** (default focus) and **Delete** (destructive, red variant). States: **idle**, **deleting** (Delete button shows spinner, both buttons disabled), **error** (dialog stays open, error message shown with retry option). On success: dialog closes, card is removed from the list column, focus moves to the column heading or the next card. `Esc` at any point cancels. WCAG 2.2 AA: `role="alertdialog"`, focus trap, `aria-describedby` on description paragraph.

## Out of scope
- Soft-delete / archive / restore (deferred — v1 is hard delete only).
- Bulk delete (deferred).
- Deleting a list or board (separate features in EPIC-2 / Boards epic).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `DeleteCardCommandHandlerTests` (Application unit — success → card removed, Viewer role → authz exception, non-member → not-found, card not found → not-found); `DeleteCardEndpointTests` (Api integration — 204 and subsequent GET returns 404, 401 no token, 403 Viewer, 404 non-member, 404 card-not-found); `DeleteCardDialog.spec.ts` (Vitest — dialog opens on trigger, Cancel closes without action, Esc cancels, confirm issues DELETE, deleting state disables buttons, error state shown with retry, Viewer sees no delete control, a11y assertions)
