# STORY-5.4: Delete a comment

**Epic:** EPIC-5

As the author of a comment or the board Owner, I want to delete a comment, so that inaccurate or inappropriate content can be removed from the discussion.

## Acceptance Criteria

**Given** I am authenticated as the **author** of a comment (with role Owner or Editor) **When** I confirm and send DELETE to `/api/boards/{boardId}/cards/{cardId}/comments/{commentId}` **Then** the comment is permanently deleted and I receive `204 No Content`.

**Given** I am authenticated as the **board Owner** (but **not** the comment's author) **When** I confirm and send DELETE **Then** the comment is permanently deleted and I receive `204 No Content` (board Owner may delete any comment on any card in that board).

**Given** I am authenticated as the **board Owner AND the comment's author** **When** I confirm and send DELETE **Then** the comment is permanently deleted and I receive `204 No Content` (the author+Owner combination satisfies both the authorship path and the board Owner path; either alone is sufficient).

**Given** the comment is successfully deleted **When** other members are connected **Then** the comment disappears in real-time (propagation < 1 s via EPIC-6) without a manual refresh.

**Given** I am authenticated as an **Editor who is not the comment's author** **When** I attempt to delete the comment **Then** I receive `403` RFC 7807 and the comment is not deleted (non-author, non-Owner editors cannot delete another user's comment).

**Given** I am authenticated as a **Viewer** (regardless of authorship) **When** I attempt to delete a comment **Then** I receive `403` RFC 7807 and the comment is not deleted.

**Given** I am **not a member** of the board **When** I attempt to delete a comment **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** I present **no or invalid JWT** **When** I attempt to delete a comment **Then** I receive `401` RFC 7807.

**Given** the `commentId` does not exist or does not belong to `cardId`/`boardId` **When** I attempt to delete **Then** I receive `404` RFC 7807.

**Given** I am the author or the board Owner viewing a comment **When** the card detail is rendered **Then** a **Delete** button/icon is visible on eligible comments (hidden from non-authors/non-Owners and Viewers); clicking it opens a **confirmation dialog** ("Delete this comment? This cannot be undone.") with Confirm and Cancel; confirming sends the DELETE request; cancelling closes the dialog without any network call; the UI meets **WCAG 2.2 AA** (modal focus trap, `role="alertdialog"`, keyboard-dismissible).

## NFR / constraints

- **Security / authz (layered):** two checks enforced server-side in order — (1) caller is a board member; (2) caller is either the comment's author OR a board Owner. Failure at (1): `404` (non-member) or `401`. Failure at (2): `403`. Board Owner may delete **any** comment; no other role may delete a comment they do not own.
- **Destructive action — confirmation required:** the UI must show a confirmation dialog before sending the DELETE request. No "undo". This applies to both the author and the board Owner paths.
- **Idempotency:** a second DELETE on an already-deleted comment returns `404` (not `204`); comment IDs are not reused.
- **API contract:** `DELETE /api/boards/{boardId}/cards/{cardId}/comments/{commentId}`. Returns `204 No Content` on success. RFC 7807 on all error paths.
- **Clean Architecture:** use-case in Application (receives `callerId`, performs authz); EF Core hard-delete in Infrastructure; thin controller in Api (no soft-delete in v1 — out of scope).
- **Cascades:** deleting a comment removes only that comment; card and board are unaffected.
- **Accessibility:** WCAG 2.2 AA — confirmation dialog has `role="alertdialog"`, `aria-labelledby`, `aria-describedby`; focus moves into dialog on open; `Esc` cancels; focus returns to trigger element on close.

## UX spec (skeleton)

Within each comment card (from STORY-5.2 list), when the current user is the comment's author **or** the board Owner and role is not Viewer: a **Delete** icon button (trash, `aria-label="Delete comment"`) is visible.

On click — **confirmation dialog** (modal):
- Title: "Delete comment?"
- Body: "This will permanently remove the comment. This cannot be undone."
- Actions: **Delete** (destructive, red) | **Cancel**.
- `Esc` or clicking outside closes and cancels.
- Focus trap inside dialog; focus restores to trigger button on close.

Submitting state: Delete button shows spinner, both buttons disabled.
Server error: error message inside dialog; dialog stays open so user can retry or cancel.
Success: dialog closes; comment removed from list; `role="status"` toast "Comment deleted."

No external comp — this note is the design reference.

## Out of scope

- Soft-delete / restore / trash functionality (hard-delete only in v1).
- Bulk delete of multiple comments.
- Deleting all comments on a card in one action.
- Audit trail of deleted comments (EPIC-7 activity feed tracks the deletion event, not this story).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.5, §1.4, §1.6, §2.2
**Tests:** `DeleteCommentHandlerTests` (Application unit — success author-Editor, success author-Owner (author who is also board Owner), success board-Owner deleting another's comment, non-author Editor → forbidden, Viewer → forbidden, non-member → not-found, commentId not belonging to card → not-found, already-deleted commentId → not-found); `DeleteCommentEndpointTests` (Api integration — 204 success author-Editor, 204 success author-Owner (dual role), 204 success board-Owner on other's comment, 403 non-author Editor, 403 Viewer, 401, 404 non-member, 404 wrong commentId, second DELETE → 404); `DeleteCommentConfirmation.spec.ts` (Vue/Vitest — delete button visible only to author/board-Owner non-Viewer, dialog opens on click, Esc cancels, confirm sends DELETE, loading state disables buttons, server error stays in dialog, success removes comment from list + toast, focus management, a11y role=alertdialog)
