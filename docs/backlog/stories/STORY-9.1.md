# STORY-9.1: View my assigned cards across all my boards (paginated, with isolation guarantee)

**Epic:** EPIC-9

As an authenticated user, I want to see every card assigned to me across all boards I am a member of, so that I have a single place to review my outstanding work regardless of which board it lives on.

## Acceptance Criteria

**Given** I am authenticated and assigned to one or more cards across one or more boards **When** I GET `/me/tasks` **Then** I receive a paginated response `{ items, page, pageSize, total }` where each item contains: `cardId`, `cardTitle`, `cardDueDate` (nullable), `cardPriority`, `boardId`, `boardName`, `listId`, `listName`; and the result includes exactly the cards assigned to me on boards where I am an active member.

**Given** I am authenticated and have no cards assigned to me **When** I GET `/me/tasks` **Then** I receive `200 OK` with `{ items: [], page: 1, pageSize: 20, total: 0 }` — not a `404`.

**Given** I am authenticated **When** I GET `/me/tasks?page=2&pageSize=10` **Then** I receive the correct page of results following the shared offset-based pagination envelope; `page` defaults to 1, `pageSize` defaults to 20 (max 100); out-of-range `page` or non-integer/negative `pageSize` returns RFC 7807 `400`.

**Given** I am a member of Board A and am assigned to a card there, AND I am NOT a member of Board B but am assigned to a card there (e.g. a test fixture with a direct DB row) **When** I GET `/me/tasks` **Then** the response NEVER contains the card from Board B — the isolation guarantee holds regardless of the assignment row's existence.

**Given** I am removed from a board (EPIC-1 STORY-1.4) that had cards assigned to me **When** I GET `/me/tasks` after removal **Then** none of the cards from that board appear in the response — the membership boundary is enforced at query time, not cached.

**Given** I am unassigned from a card (EPIC-4 STORY-4.3) **When** I GET `/me/tasks` after the unassignment **Then** that card no longer appears in the response.

**Given** I present no token or an invalid/expired token **When** I GET `/me/tasks` **Then** I receive `401 Unauthorized`.

**Given** I open the My Tasks view **When** results are loading, there are no assignments, or an error occurs **Then** a loading state, a friendly empty state ("Nothing assigned to you yet"), and an error state are each shown appropriately; all states meet WCAG 2.2 AA.

## NFR / constraints
- **Security / isolation:** the query MUST join through the user's active board membership — not merely the assignee table alone. Any card on a board the user cannot access must be excluded. This is the primary security invariant of this feature.
- **Performance:** the assignee → card → list → board join is backed by a database index on `(UserId, CardId)` in the assignee join table (or equivalent composite) and a covering index on board membership `(UserId, BoardId)` to meet the <1.5 s response-time NFR (§1.6) for a user with up to the expected card volume.
- **Pagination:** offset-based; `page` (1-indexed, default 1), `pageSize` (default 20, max 100); `{ items, page, pageSize, total }` — consistent with every other list endpoint.
- **API:** DTOs only at the boundary; RFC 7807 on every error path; `401` unauthenticated; `400` invalid query params.
- **Clean Architecture (§2.2):** `GetMyTasksQuery` use-case in Application; repository/query in Infrastructure; thin controller action in Api. No EF/Microsoft.* references in Domain.
- **Due dates:** stored as UTC `DateTimeOffset`; surfaced as-is in the DTO; overdue rendering is a client concern.

## UX spec (skeleton)
Dedicated **My Tasks** page, accessible from the global navigation (e.g. top nav link). Header: "My Tasks" + total count badge. Card list rendered as a vertical feed of task rows; each row shows: **card title** (truncated at ~80 chars with full title in tooltip), **board name** chip, **list name** (secondary text), **priority badge** (`None` hidden; `Low`/`Medium`/`High` as colored chips), **due date** (formatted relative or absolute; overdue shown in a warning color). States: loading skeleton (rows); empty state illustration + copy ("Nothing assigned to you yet — cards assigned to you across your boards will appear here"); error banner (`role="alert"` with retry action). Pagination controls: Previous / Next + page indicator. WCAG 2.2 AA: all interactive elements keyboard-accessible; priority badges and due-date warnings do not rely solely on color (text label always present); task rows have accessible names (`aria-label` including card title and board name).

## Out of scope
- Sorting or filtering the list (STORY-9.2).
- Navigating to the card's board view (STORY-9.3).
- Mutating the card from this view (EPIC-4).
- Real-time push of new assignments into this view (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.5, §1.6, §2.2
**Tests:** `GetMyTasksHandlerTests` (Application unit — returns only caller's assignments on boards where caller is a member; excludes cards from boards caller is not a member of; excludes unassigned cards; returns empty list when no assignments; pagination offset/limit applied correctly); `MyTasksIsolationTests` (Application/Infrastructure integration — card on a non-member board does NOT appear even with a direct assignee row; card disappears after board membership is removed; card disappears after unassignment); `MyTasksEndpointTests` (Api integration — 200 with paginated shape, 401 unauthenticated, 400 bad page params, empty-list 200 not 404); `MyTasksView.spec.ts` (Vue/Vitest — loading/empty/error states, row content rendering, pagination controls, a11y: accessible row names, priority not color-only, due date not color-only)
