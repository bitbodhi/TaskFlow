# STORY-5.1: Add a comment to a card

**Epic:** EPIC-5

As an Editor or Owner of a board, I want to post a comment on a card, so that I can leave context, questions, or decisions directly on the work item.

## Acceptance Criteria

**Given** I am authenticated as an Editor or Owner of the board that contains the card **When** I POST `{ body }` (1–5000 characters) to `/api/boards/{boardId}/cards/{cardId}/comments` **Then** the comment is persisted and I receive `201 Created` with a comment DTO (`id`, `cardId`, `authorId`, `authorDisplayName`, `body`, `createdAt`, `updatedAt`, `isEdited: false`).

**Given** the comment is successfully created **When** other members are connected **Then** they observe the new comment appear in real-time (propagation < 1 s via EPIC-6) without refreshing.

**Given** I am authenticated as a **Viewer** of the board **When** I POST a comment **Then** I receive `403` RFC 7807 and no comment is created (server-side role check — Viewers are read-only per §1.4).

**Given** I am **not a member** of the board **When** I POST a comment **Then** I receive `404` RFC 7807 (board/card existence not disclosed to non-members) and no comment is created.

**Given** I present **no or invalid JWT** **When** I POST a comment **Then** I receive `401` RFC 7807 and no comment is created.

**Given** I submit a body that is **empty or exceeds 5000 characters** **When** I POST a comment **Then** I receive `400` RFC 7807 with a validation problem detail and no comment is created.

**Given** the `cardId` does not belong to `boardId` (cross-board insertion attempt) **When** I POST a comment **Then** I receive `404` RFC 7807 and no comment is created (card–board membership verified server-side to prevent IDOR).

**Given** I am on the card detail view **When** I open the comment panel **Then** I see an add-comment text area (1–5000 chars), a submit button, and inline character count; loading, validation-error, and server-error states are all present; the form meets **WCAG 2.2 AA** (label, `aria-describedby` for errors, keyboard-only operable).

## NFR / constraints

- **Security / authz:** role check against board membership performed server-side before any persistence. Roles that may create comments: `Owner`, `Editor`. `Viewer` → `403`. Non-member → `404`. Unauthenticated → `401`.
- **Validation:** body length 1–5000 characters enforced in the Domain `Comment` entity, the EF column (`nvarchar(5000) NOT NULL`), and client-side (non-blocking, server is authoritative).
- **API contract:** `POST /api/boards/{boardId}/cards/{cardId}/comments`. Returns `201` with comment DTO. RFC 7807 on all error paths.
- **DTOs only** at the API boundary; no EF entities exposed.
- **Clean Architecture:** `Comment` entity in Domain; use-case in Application; EF Core persistence in Infrastructure; thin controller in Api (no EF/Microsoft.* in Domain or Application layers).
- **Performance:** comment creation must complete in < 500 ms under normal load (DB write + SignalR publish).
- **Reliability:** EF Core migration adds `Comments` table with FK to `Cards` and `AspNetUsers`; structured logging; global RFC 7807 error handler.
- **Accessibility:** WCAG 2.2 AA on the add-comment form; errors use `role="alert"` and `aria-describedby`.

## UX spec (skeleton)

Card detail panel — "Comments" section at the bottom. A **text area** labelled "Add a comment" with a live character counter (`n / 5000`). A **Submit** button (disabled while empty or over limit). States:
- **Empty:** placeholder text "Leave a comment…"; submit disabled.
- **Loading / submitting:** submit shows spinner, text area disabled.
- **Validation error (client):** inline error below text area via `aria-describedby`; character count turns red when limit exceeded.
- **Server error:** `role="alert"` banner above the form.
- **Success:** text area cleared; new comment prepended/appended to list (see STORY-5.2 for rendering).
Keyboard: `Tab` reaches text area and submit; `Ctrl+Enter` submits; `Esc` clears text area. No external comp — this note is the design reference.

## Out of scope

- Editing or deleting comments (STORY-5.3, STORY-5.4).
- @mention autocomplete.
- Rich-text / markdown input.
- File attachments on comments.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.5, §1.4, §1.6, §2.2
**Tests:** `AddCommentHandlerTests` (Application unit — success Owner, success Editor, Viewer → forbidden, non-member → not-found, empty body → validation, body > 5000 chars → validation, cross-board cardId → not-found); `AddCommentEndpointTests` (Api integration — 201 DTO shape, 400 RFC 7807, 401, 403 Viewer, 404 non-member, 404 cross-board card, persistence read-back); `AddCommentForm.spec.ts` (Vue/Vitest — character counter, submit disabled when empty/over-limit, loading state, inline validation error, server-error banner, a11y attributes)
