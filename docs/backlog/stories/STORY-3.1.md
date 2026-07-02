# STORY-3.1: View card details

**Epic:** EPIC-3

As a board member (any role), I want to open a card and see its title, description, and the list and board it belongs to, so that I can understand the full context of the task.

## Acceptance Criteria

**Given** I am an authenticated member of the board (Owner, Editor, or Viewer) **When** I GET `/cards/{cardId}` **Then** I receive `200 OK` with a card DTO containing: `id`, `title`, `description` (nullable), `listId`, `listName`, `boardId`, `boardName`, `rank`, `createdAt`, `updatedAt`.

**Given** I am not a member of the board that owns the card, or I present no or an invalid JWT **When** I GET `/cards/{cardId}` **Then** I receive `404` RFC 7807 (non-member — existence not disclosed) or `401` (unauthenticated), respectively.

**Given** `cardId` does not exist **When** I request it **Then** I receive `404` RFC 7807 — indistinguishable from the non-member 404 (no information disclosure).

**Given** I am a board member **When** I open the card detail view in the UI **Then** the panel/modal displays: card title, description (or placeholder if empty), the containing list name, and the board name. Loading, empty-description, and error states are all shown correctly. The view is read-only for Viewers.

**Given** the card detail view loads **When** it renders **Then** it meets WCAG 2.2 AA: heading hierarchy correct, focus moves into the panel on open, `Esc` closes it, all interactive controls are keyboard-accessible with visible focus indicators.

## NFR / constraints
- API response is a **DTO only** at the boundary (no domain or EF types leak).
- Server-side authorization: membership is checked before any card data is returned.
- No separate list-of-cards endpoint is added here — that is already covered by EPIC-2 (list cards) and EPIC-0 (board card feed, paginated). This story adds only the single-card detail endpoint.
- Real-time propagation of card edits made by others is a constraint: changes made by a concurrent user will not appear until the user reopens the card or a SignalR push arrives (EPIC-6 concern — not authored here).

## UX spec (skeleton)
Card detail opens as a side-panel or modal (design choice deferred to Figma; skeleton here). Sections: **title** (read-only label), **description** (read-only text area or rendered text; "No description" placeholder when empty), **metadata strip** (List: `{listName}` · Board: `{boardName}`). States: **loading** (skeleton shimmer), **error** (alert banner with retry), **empty description** (muted placeholder text). Panel has a close button (×) and traps focus. `Esc` closes. On mobile: full-screen sheet. No destructive action on this view (edit/delete are separate stories). WCAG 2.2 AA: `role="dialog"`, `aria-modal="true"`, `aria-labelledby` pointing to title heading.

## Out of scope
- Editing or deleting the card (STORY-3.2, STORY-3.3).
- Card metadata: labels, assignees, due dates, priorities (EPIC-4).
- Comments section (EPIC-5).
- Activity log within the card (EPIC-7).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.5, §1.6, §2.2
**Tests:** `GetCardQueryHandlerTests` (Application unit — success with full DTO fields, non-member returns null → 404, unauthenticated → 401, card not found → 404); `GetCardEndpointTests` (Api integration — 200 DTO shape, 401 no token, 404 non-member, 404 card-not-found indistinguishable); `CardDetailPanel.spec.ts` (Vitest — loading/error/empty-description states, focus trap, Esc closes, a11y assertions via axe-core)
