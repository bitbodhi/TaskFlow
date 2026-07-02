# STORY-9.2: Sort and filter My Tasks for triage

**Epic:** EPIC-9

As an authenticated user, I want to sort my assigned cards by due date or priority and filter them by board or priority, so that I can triage my workload and focus on what matters most.

## Acceptance Criteria

**Given** I am authenticated **When** I GET `/me/tasks?sortBy=dueDate&sortDir=asc` **Then** the paginated response is ordered by `cardDueDate` ascending; cards with no due date appear last.

**Given** I am authenticated **When** I GET `/me/tasks?sortBy=dueDate&sortDir=desc` **Then** the response is ordered by `cardDueDate` descending; cards with no due date appear last.

**Given** I am authenticated **When** I GET `/me/tasks?sortBy=priority&sortDir=desc` **Then** the response is ordered by priority descending (`High` → `Medium` → `Low` → `None`).

**Given** I am authenticated **When** I GET `/me/tasks?sortBy=priority&sortDir=asc` **Then** the response is ordered by priority ascending (`None` → `Low` → `Medium` → `High`).

**Given** I am authenticated **When** I supply an unrecognised `sortBy` value or an invalid `sortDir` value **Then** I receive RFC 7807 `400` and no results.

**Given** I am authenticated **When** I GET `/me/tasks?boardId={boardId}` **Then** the response contains only cards from that specific board (still within my membership — a `boardId` I am not a member of returns an empty list, not a `404` or a leak).

**Given** I am authenticated **When** I GET `/me/tasks?priority=High` **Then** the response contains only cards with `cardPriority = High`; invalid priority values return RFC 7807 `400`.

**Given** I supply a `boardId` value that is not a valid GUID (e.g. `"abc"`, `"123"`, an empty string) **When** I GET `/me/tasks?boardId={value}` **Then** I receive `400` RFC 7807 with a field-level problem detail identifying the `boardId` parameter; no results are returned. (A syntactically valid GUID that is not a board I am a member of still returns `200` with an empty list — the board's existence is not disclosed.)

**Given** I supply both `boardId` and `priority` filter params **When** I GET `/me/tasks` **Then** both filters are applied together (AND logic); sorting is also applied if provided.

**Given** I omit `sortBy` **When** I GET `/me/tasks` **Then** the default ordering is by `cardDueDate` ascending with nulls last (consistent with STORY-9.1's default presentation).

**Given** I open the My Tasks view **When** I interact with the sort/filter controls **Then** the view updates to reflect the selected options; loading, empty, and error states each render correctly for the filtered/sorted result; all controls meet WCAG 2.2 AA.

## NFR / constraints
- **Isolation inherited:** sort and filter are applied on top of the membership-gated query from STORY-9.1. Filtering by `boardId` never bypasses the membership check — even if the caller supplies a `boardId` they are not a member of, the result is an empty list (the board's existence is not disclosed).
- **Supported sort keys:** `dueDate`, `priority`. Default: `dueDate` asc, nulls last. `sortDir`: `asc` | `desc`.
- **Supported filter params:** `boardId` (must be a syntactically valid GUID — invalid GUID format returns `400` RFC 7807; a valid GUID that is not a board the caller is a member of returns an empty list, not `404`); `priority` (`None` | `Low` | `Medium` | `High`).
- **Combining:** sort and filter params are combined additively. Pagination applies after filtering and sorting.
- **Performance:** the same assignee/membership indexes from STORY-9.1 are sufficient; filter params map to `WHERE` clause predicates rather than application-side filtering.
- **API:** query-string params only (no request body on GET); RFC 7807 `400` for unrecognised/invalid param values; DTOs unchanged from STORY-9.1.
- **Clean Architecture:** `GetMyTasksQuery` extended with optional sort/filter fields; the Application use-case assembles the predicate; Infrastructure translates to EF Core query; no raw SQL unless a query-plan analysis requires it.

## UX spec (skeleton)
My Tasks page (established in STORY-9.1) gains a **toolbar** below the header: a **Sort by** dropdown (`Due date ↑`, `Due date ↓`, `Priority ↑`, `Priority ↓`; default: Due date ↑); a **Board** filter dropdown (list of boards the user is a member of + "All boards" default); a **Priority** filter chip-group (`All`, `High`, `Medium`, `Low`). Controls are placed in a single horizontal row that wraps on narrow viewports. Selecting any control re-fetches from page 1. Active filters show a "Clear filters" text button. States: while re-fetching, the list shows a loading skeleton overlay (not a full page spinner); empty state when filters produce no results ("No tasks match your filters"); error banner on failure. WCAG 2.2 AA: all dropdowns and chip-group items are keyboard-navigable; active filter state is announced via `aria-pressed` (chip-group) or `aria-selected` (dropdown option); "Clear filters" is focusable and labelled.

## Out of scope
- Filtering by label within My Tasks (future cross-epic concern).
- Persisting filter/sort preferences across sessions (future).
- Full-text search of card titles from this view (EPIC-8 covers within-board search; cross-board text search is future).
- Saved filter presets.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.5, §1.6, §2.2
**Tests:** `GetMyTasksHandlerSortFilterTests` (Application unit — dueDate asc/desc nulls-last, priority asc/desc, boardId filter stays within membership, non-member boardId returns empty not error, invalid-GUID boardId → 400 RFC 7807, priority filter, combined sort+filter, invalid sortBy/sortDir/priority → validation error); `MyTasksSortFilterEndpointTests` (Api integration — 200 with correct ordering for each sort combo, 200 empty for non-member boardId (valid GUID), 400 for invalid-GUID boardId RFC 7807, 400 for other invalid params, combined params applied correctly); `MyTasksToolbar.spec.ts` (Vue/Vitest — sort dropdown selection triggers re-fetch at page 1, board filter, priority chips, clear-filters resets all, loading overlay during re-fetch, empty/error states with active filters, a11y: aria-pressed/aria-selected, keyboard navigation)
