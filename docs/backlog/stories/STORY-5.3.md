# STORY-5.3: Edit own comment

**Epic:** EPIC-5

As the author of a comment, I want to edit the body of my comment, so that I can correct mistakes or add information after posting.

## Acceptance Criteria

**Given** I am authenticated as the **author** of a comment and an active board member (Owner or Editor) **When** I PUT `{ body }` (1–5000 characters) to `/api/boards/{boardId}/cards/{cardId}/comments/{commentId}` **Then** the comment body is updated, `updatedAt` is refreshed, and I receive `200 OK` with the updated comment DTO (same shape as STORY-5.1, `isEdited: true`).

**Given** the comment is successfully edited **When** other members are connected **Then** they observe the updated body and "Edited" indicator in real-time (propagation < 1 s via EPIC-6).

**Given** I am authenticated but I am **not the author** of the comment **and** I am **not the board Owner** **When** I attempt to edit the comment **Then** I receive `403` RFC 7807 and the comment is not modified (ownership rule: only the author may edit their own comment; board Owner cannot edit another's comment, only delete it per STORY-5.4).

**Given** I am authenticated as a **Viewer** of the board (even if I were somehow the original author via a role change) **When** I attempt to edit a comment **Then** I receive `403` RFC 7807 — Viewer role forbids all mutations.

**Given** I am **not a member** of the board **When** I attempt to edit a comment **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** I present **no or invalid JWT** **When** I attempt to edit a comment **Then** I receive `401` RFC 7807.

**Given** I submit an **empty body or one exceeding 5000 characters** **When** I attempt to edit **Then** I receive `400` RFC 7807 with a validation problem detail; the comment is not modified.

**Given** the `commentId` does not exist or does not belong to `cardId`/`boardId` **When** I attempt to edit **Then** I receive `404` RFC 7807.

**Given** the `cardId` does not belong to `boardId` **When** I attempt to edit a comment **Then** I receive `404` RFC 7807 (existence not disclosed — matching STORY-5.1's explicit cross-board guard).

**Given** I send a PUT request body that includes fields other than `body` (e.g. `authorId`, `createdAt`) **When** the server processes the request **Then** those extra fields are silently ignored; only `body` (1–5000 characters) is accepted and updated; all other comment fields remain immutable.

**Given** I am viewing a comment I authored **When** the card detail is rendered **Then** an **Edit** button/icon is visible on my comment (hidden from non-authors and Viewers); clicking it opens an inline edit form pre-populated with the current body; submitting saves and shows the updated text plus an "Edited" badge; cancelling restores the original text without a network call; form meets **WCAG 2.2 AA**.

## NFR / constraints

- **Security / authz (layered):** two checks enforced server-side in order — (1) caller is a board member with role `Owner` or `Editor`; (2) caller is the comment's author. Failure at (1): `403` (Viewer) or `404` (non-member) or `401`. Failure at (2): `403`. The board Owner does **not** have permission to edit another user's comment (edit = author-only; delete = author or board Owner).
- **Request body contract:** the PUT endpoint accepts `{ body }` ONLY (1–5000 characters). Fields such as `authorId`, `createdAt`, `updatedAt`, `isEdited`, or any other comment property are immutable — they are silently ignored even if present in the request body; the server never updates them via this endpoint.
- **Validation:** body 1–5000 characters, same constraint as STORY-5.1.
- **cardId scope:** a `cardId` that does not belong to `boardId` returns `404` (existence not disclosed — consistent with STORY-5.1's cross-board guard).
- **Idempotency:** re-submitting the same body is allowed and returns `200` (no "nothing changed" error); `isEdited` is set to `true` and `updatedAt` updated on every successful edit.
- **API contract:** `PUT /api/boards/{boardId}/cards/{cardId}/comments/{commentId}`. Returns `200` with updated DTO. RFC 7807 on all error paths.
- **DTOs only** at the boundary.
- **Clean Architecture:** use-case in Application (receives `callerId`, performs authz); EF Core update in Infrastructure; thin controller in Api.
- **Optimistic locking:** not required in v1 (last-write-wins acceptable for comments).
- **Accessibility:** WCAG 2.2 AA — inline edit form has visible label, character counter, error via `aria-describedby`; cancel action keyboard-accessible (`Esc` cancels).

## UX spec (skeleton)

Within each comment card (from STORY-5.2 list), when the current user is the author and has role Owner or Editor: an **Edit** icon button (pencil, `aria-label="Edit comment"`) is visible. On click:
- Comment body text is replaced with a **text area** pre-filled with current body; character counter shown.
- **Save** button (disabled if unchanged or invalid) and **Cancel** link.
- **Submitting** state: Save shows spinner, text area disabled.
- **Validation error:** inline below text area via `aria-describedby`.
- **Server error:** `role="alert"` banner within the comment card.
- **Success:** inline editor collapses; body shows updated text; "Edited" badge appears next to timestamp.
- `Esc` cancels without saving. No external comp — this note is the design reference.

## Out of scope

- Editing another user's comment (that is a `403`, not a feature).
- Edit history / revision log (deferred to future).
- Markdown/rich-text editing.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.5, §1.4, §1.6, §2.2
**Tests:** `EditCommentHandlerTests` (Application unit — success author-Owner, success author-Editor, non-author non-Owner → forbidden, non-author board-Owner → forbidden (board Owner cannot edit others), Viewer author → forbidden, non-member → not-found, invalid body empty → validation, body > 5000 → validation, commentId not belonging to card → not-found, cardId not belonging to boardId → not-found, extra fields in body are ignored and do not mutate authorId/createdAt, isEdited becomes true, updatedAt refreshed); `EditCommentEndpointTests` (Api integration — 200 updated DTO, isEdited true, 400, 401, 403 non-author, 403 Viewer, 404 non-member, 404 wrong commentId, 404 cardId from different board, PUT with extra fields returns 200 but extra fields unchanged in response); `EditCommentForm.spec.ts` (Vue/Vitest — edit button only visible to author non-Viewer, inline form prefilled, Esc cancels, submit disabled when unchanged/invalid, loading state, error banner, success shows Edited badge, a11y)
