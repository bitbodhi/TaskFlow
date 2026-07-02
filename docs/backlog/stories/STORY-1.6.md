# STORY-1.6: Delete a board (Owner only)

**Epic:** EPIC-1

As a board Owner, I want to permanently delete a board after confirming the action, so that I can remove projects that are no longer needed and clean up all associated data.

## Acceptance Criteria

**Given** I am a board Owner **When** I confirm deletion and DELETE `/boards/{boardId}` **Then** the board, all its lists, all cards on those lists, all memberships, all labels, and all comments are deleted (cascade), I receive `204 No Content`, and subsequent `GET /boards/{boardId}` returns `404` for all users.

**Given** I am a board Owner **When** I DELETE the board and the cascade completes **Then** the board no longer appears in any member's board list (`GET /boards`), including for users who were previously Editors or Viewers.

**Given** I am an Editor or Viewer **When** I DELETE `/boards/{boardId}` **Then** I receive `403` RFC 7807 (Owner-only action) and the board is not deleted.

**Given** I am unauthenticated **When** I DELETE `/boards/{boardId}` **Then** I receive `401` RFC 7807.

**Given** I am authenticated but not a member of the board **When** I DELETE `/boards/{boardId}` **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** I initiate board deletion via the UI **When** I click the "Delete board" action **Then** a confirmation dialog appears with the board name ("Delete board 'My Project'? This will permanently delete all lists, cards, and members. This action cannot be undone.") with a destructive-confirm button and a cancel button before any request is sent.

**Given** I confirm deletion **When** the request is in flight **Then** the confirm button is disabled/spinner shown, preventing double-submission.

**Given** deletion succeeds **When** the response is `204` **Then** I am navigated away from the (now-deleted) board to the boards list, a `role="status"` announcement confirms deletion, and the board does not appear in the list.

**Given** deletion fails (5xx) **When** the dialog is still open **Then** an error message is shown within the dialog (`role="alert"`), the dialog remains open, and the board is not removed from the UI.

**Given** the confirm dialog and delete action are rendered **When** used by keyboard or assistive technology **Then** focus is trapped within the dialog when open, focus returns to a sensible element (boards list heading or first board) on close, and the dialog meets WCAG 2.2 AA.

## NFR / constraints

- **Security / authorization:** Owner-only; `403` for Editor/Viewer; `404` for non-members; `401` unauthenticated (per STORY-1.5 role matrix).
- **Cascade:** deletion must cascade in the **database** (EF `DeleteBehavior.Cascade` or SQL ON DELETE CASCADE) so that no orphaned rows remain. The cascade order: members → comments → card labels → card assignees → cards → lists → labels → board. A single transaction must cover the whole cascade.
- **Destructive action:** confirmation dialog mandatory before the DELETE request is sent; no undo.
- **API:** RFC 7807 on every error path; `204 No Content` on success (no body).
- **Clean Architecture:** `DeleteBoardCommand` in Application; cascade persistence in Infrastructure; no EF/Microsoft.* types in Domain.
- **Real-time:** board deletion is propagated to all connected members <1s (via EPIC-6), redirecting them to the boards list — this story does not author SignalR code.
- **Accessibility:** WCAG 2.2 AA; confirm dialog focus trap; destructive button styled distinctively; board name shown in dialog to prevent misidentification.

## UX spec (skeleton)

Board settings menu (accessible from the board header, e.g. a "..." or gear icon — Owner-only, not shown to Editor/Viewer): one option is "Delete board". Activating it opens a **confirm dialog**: title "Delete board", body "Delete '[Board Name]'? All lists, cards, labels, comments, and members will be permanently deleted. This action cannot be undone.", two buttons — "Delete board" (destructive) and "Cancel". **Dialog states:** **idle** (both buttons enabled), **submitting** (Delete button disabled/spinner, Cancel disabled), **error** (`role="alert"` inline message). On success: dialog closes, navigate to `/boards`, `role="status"` announcement "Board deleted". Keyboard: Esc cancels; Tab cycles in dialog (focus trap); on close, focus moves to board list.

## Out of scope

- Soft-delete / archive / restore (permanent deletion only in v1).
- Notifying other members that the board was deleted (future notification system).
- Exporting board data before deletion (future).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `DeleteBoardHandlerTests` (Application unit — success cascade verified, caller is Editor → 403, caller is non-member → 404); `DeleteBoardEndpointTests` (Api integration — 204, 401, 403, 404, board + lists + cards + members + labels + comments absent after deletion in a single transaction); `DeleteBoardDialog.spec.ts` (Vue/Vitest — dialog open/cancel/confirm, board name in dialog body, submitting state, error state, success navigation, focus trap, a11y); Owner-only visibility of delete action in UI tested in `BoardSettingsMenu.spec.ts`
