# STORY-5.2: View a card's comments (paginated)

**Epic:** EPIC-5

As any board member (Owner, Editor, or Viewer), I want to see all comments on a card in chronological order, so that I can follow the discussion without losing context.

## Acceptance Criteria

**Given** I am authenticated as **any board member** (Owner, Editor, or Viewer) **When** I GET `/api/boards/{boardId}/cards/{cardId}/comments` **Then** I receive `200 OK` with a paginated envelope `{ items, page, pageSize, total }` containing comment DTOs in ascending `createdAt` order; each DTO includes (`id`, `cardId`, `authorId`, `authorDisplayName`, `body`, `createdAt`, `updatedAt`, `isEdited`).

**Given** the card has no comments **When** I GET the comments list **Then** I receive `200 OK` with `{ items: [], page: 1, pageSize: 20, total: 0 }` (not `404`).

**Given** I provide valid `page` and `pageSize` query parameters **When** I GET the comments list **Then** the response honors offset-based pagination: `page` is 1-indexed (default `1`), `pageSize` default `20` max `100`; `total` reflects the full unfiltered count; requesting a page beyond the last returns `{ items: [], page: N, pageSize, total }`.

**Given** I supply an invalid `pageSize` (e.g. `0`, negative, or > 100) **When** I GET the comments list **Then** I receive `400` RFC 7807 with a validation problem detail.

**Given** I am **not a member** of the board **When** I GET the comments list **Then** I receive `404` RFC 7807 (existence not disclosed).

**Given** I present **no or invalid JWT** **When** I GET the comments list **Then** I receive `401` RFC 7807.

**Given** the `cardId` does not belong to `boardId` **When** I GET the comments list **Then** I receive `404` RFC 7807.

**Given** I am on the card detail view **When** comments are loading **Then** a skeleton/loading state is shown; **When** there are no comments, an empty state ("No comments yet") is shown; **When** the fetch fails, a server-error banner with a retry action is shown; all states meet **WCAG 2.2 AA**.

**Given** a new comment is posted by another user **When** I am viewing the card **Then** it appears in the list in real-time (propagation < 1 s via EPIC-6) without a manual refresh.

## NFR / constraints

- **Security / authz:** `GET` is permitted for **all board members** including Viewers; non-member → `404`; unauthenticated → `401`. Server-side only.
- **Pagination:** offset-based, identical envelope `{ items, page, pageSize, total }` as established in EPIC-0 (STORY-0.4). `pageSize` max 100, default 20; `page` 1-indexed, default 1.
- **Ordering:** results ordered by `createdAt ASC` (chronological). Stable sort required (tie-break on `id ASC`).
- **Performance:** comment list for a card (up to 100 per page) returns in < 500 ms; indexed on `CardId, CreatedAt`.
- **API contract:** `GET /api/boards/{boardId}/cards/{cardId}/comments?page=1&pageSize=20`. Returns `200` with pagination envelope. RFC 7807 on all error paths.
- **DTOs only** at the boundary; `isEdited` is `true` when `updatedAt > createdAt`.
- **Clean Architecture:** query in Application; EF Core read in Infrastructure; thin controller in Api.
- **Accessibility:** WCAG 2.2 AA — loading/empty/error states; comment list is a semantic list (`<ul>`/`<li>`) with accessible author attribution.

## UX spec (skeleton)

Card detail panel — "Comments" section. A **list** of comment cards, each showing: author avatar/initials, author display name, relative timestamp (e.g. "2 hours ago"), body text, and an "Edited" badge if `isEdited`. States:
- **Loading:** skeleton cards (2–3 placeholder rows, aria-hidden).
- **Empty:** illustration + "No comments yet — be the first to comment" (hidden from Viewers who cannot post).
- **Error:** `role="alert"` banner with "Failed to load comments" + Retry button.
- **Populated:** list scrollable; **Load more** button or infinite-scroll trigger at bottom when `total > items.length`. Page/total count shown ("Showing 20 of 47 comments").
Keyboard: each comment is reachable by `Tab`; edit/delete actions (STORY-5.3/5.4) are within each comment card. No external comp — this note is the design reference.

## Out of scope

- Editing or deleting comments from this view (STORY-5.3, STORY-5.4 author the controls, but this story establishes the list rendering they appear within).
- Sorting order controls (chronological ascending is fixed in v1).
- Full-text search or filtering within a card's comments.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.5, §1.4, §1.6, §2.2
**Tests:** `GetCardCommentsHandlerTests` (Application unit — success Owner, success Editor, success Viewer, non-member → not-found, cross-board card → not-found, invalid pageSize → validation, empty list, pagination math); `GetCardCommentsEndpointTests` (Api integration — 200 envelope shape, ordering ASC, page/pageSize defaults and overrides, pageSize > 100 → 400, 401, 404 non-member, 404 cross-board card, empty list 200 not 404); `CommentList.spec.ts` (Vue/Vitest — loading skeleton, empty state, error+retry, list renders authorDisplayName+body+isEdited badge, pagination trigger)
