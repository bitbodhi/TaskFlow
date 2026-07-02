# STORY-2.5: View a board's lists in order

**Epic:** EPIC-2

As any board member (Owner, Editor, or Viewer), I want to see the board's lists rendered in their correct order when I open the board, so that I can navigate the board's column layout and find the right place to work.

## Acceptance Criteria

**Given** I am a member of a board (any role) **When** I GET the board's lists **Then** I receive the lists sorted ascending by `position` (rank), with each list DTO containing at minimum: id, boardId, name, position, createdAt.

**Given** the board has no lists **When** I GET the board's lists **Then** I receive an empty `items` array with `total: 0` and the paginated envelope is still present.

**Given** the board has lists that have been reordered (STORY-2.4) **When** I GET the board's lists **Then** the order matches the persisted rank order — reorder correctness is verified here end-to-end.

**Given** I am an unauthenticated caller **When** I GET a board's lists **Then** I receive `401`.

**Given** I am authenticated but not a member of the board **When** I GET its lists **Then** I receive `404` (existence not disclosed to non-members).

**Given** the board view is loading **When** the API call is in flight **Then** placeholder skeleton columns are shown (not a blank screen).

**Given** the board has no lists yet **When** the empty state is displayed **Then** an empty-state message ("No lists yet — add your first") is shown and an "Add list" affordance (Owner/Editor) or a read-only notice (Viewer) is visible.

**Given** the API call fails **When** the board view renders **Then** an error state with a retry option is shown and the error is announced via `role="alert"`.

**Given** the board view meets WCAG 2.2 AA **Then** list columns are navigable by keyboard, column names are labelled, loading states are announced via `aria-live`, and colour is not the sole differentiator for any state.

## NFR / constraints

- **Pagination:** the endpoint uses the shared offset-based envelope — `page` (1-indexed, default 1), `pageSize` (default 20, max 100), response `{ items, page, pageSize, total }`. In practice boards have far fewer than 20 lists; a single page is the common path. The client SHOULD request page 1 with a generous `pageSize` (e.g. 50) on initial load. No server-side streaming — full-page fetch per board open.
- **Ordering:** SQL `ORDER BY Position ASC` in the query; ties are broken by `CreatedAt ASC` (deterministic but should never occur after STORY-2.4's model is in place).
- **Security / authorization:** any authenticated member (Owner, Editor, Viewer) may read; non-member → `404`; unauthenticated → `401`. No `403` is reachable on a GET.
- **Performance (§1.6):** the board (~200 cards spread across lists) must load in < 1.5 s. This endpoint returns lists only (no nested cards); cards are fetched per-list or in a separate call. The lists query should be fast: indexed on `(BoardId, Position)`.
- **API:** `GET /boards/{boardId}/lists?page=1&pageSize=50`; DTOs only; RFC 7807 on every error path.
- **Clean Architecture (§2.2):** list-query use-case in Application (returns paginated DTO); EF Core read in Infrastructure; thin controller in Api.

## UX spec (skeleton)

Board view renders an **ordered row of list columns**, each column headed by the list name and containing its cards (cards rendered per EPIC-3). On load a **skeleton** of 2–3 grey column placeholders appears while the fetch is in progress (`aria-busy="true"` on the board container). When loaded with lists: columns appear in rank order; an **"Add list"** button appears after the last column (hidden for Viewers). When loaded with no lists: full-width **empty state** ("No lists yet") with "Add list" button (Owner/Editor) or "No lists have been created yet" (Viewer). On fetch error: full-width error panel with message and **Retry** button (`role="alert"`). Keyboard: each column header is focusable; Tab moves across columns. WCAG 2.2 AA throughout. No external comp — this note is the design reference.

## Out of scope

- Fetching cards nested inside the list response (separate card endpoint, EPIC-3).
- Infinite scroll / virtual scrolling of lists (500-list cap means pagination is a safety valve, not a UX flow).
- Filtering or sorting lists by attributes other than position.
- Real-time delivery of list additions/reorders to connected clients (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.5, §1.6, §2.2
**Tests:** `GetBoardListsHandlerTests` (Application unit — returns ordered lists, empty board, pagination envelope, authz denied non-member); `GetBoardListsEndpointTests` (Api integration — 200 paginated DTO sorted by position, empty board `total: 0`, 401, 404 non-member, ordering matches after reorder via STORY-2.4); `BoardView.spec.ts` (Vue/Vitest — skeleton loading state, empty state message, error state with retry, column order matches API response, Viewer sees no Add-list button, a11y)
