# STORY-8.1: Filter a board's cards by metadata

**Epic:** EPIC-8

As a board member, I want to filter a board's cards by one or more metadata attributes (label, assignee, priority, due-date range), so that I can focus on the work most relevant to me without scrolling through all cards.

## Acceptance Criteria

**Given** I am a member of a board **When** I call `GET /boards/{boardId}/cards?labelIds=…&assigneeIds=…&priority=…&dueBefore=…&dueAfter=…` with any combination of valid filter params **Then** I receive `200` with a paginated envelope `{ items, page, pageSize, total }` containing only cards that match ALL supplied filters (implicit AND); unmatched cards are excluded; the response is scoped to that board only.

**Given** I am a member of a board **When** I supply multiple values for `labelIds` or `assigneeIds` (comma-separated or repeated params) **Then** cards matching ANY of the supplied values for that attribute are included (OR within attribute, AND across attributes); the semantics are documented in the API response headers or an OpenAPI description.

**Given** I supply a `priority` value that is not one of the defined enum values (`None`, `Low`, `Medium`, `High`) **When** I call the filter endpoint **Then** I receive `400` RFC 7807 with a field-level problem detail identifying the `priority` parameter; no cards are returned.

**Given** I supply a `dueBefore` or `dueAfter` date **When** either is not a valid ISO 8601 date string, or `dueAfter` is later than `dueBefore` **Then** I receive `400` RFC 7807 with a clear field-level problem detail and no cards are returned.

**Given** I supply an `assigneeId` or `labelId` that does not belong to this board **When** I call the filter endpoint **Then** I receive `400` RFC 7807 (unknown filter value — not silently ignored); the error identifies the offending parameter.

**Given** all filter params are valid but no cards match **When** I call the filter endpoint **Then** I receive `200` with `{ items: [], page: 1, pageSize: <requested>, total: 0 }` — an empty result is not an error.

**Given** I am not a member of the board (or the board does not exist) **When** I call the filter endpoint **Then** I receive `404` RFC 7807; the board's existence is not disclosed.

**Given** I present no token or an expired token **When** I call the filter endpoint **Then** I receive `401` RFC 7807.

**Given** I supply a `page` or `pageSize` param **When** `page` ≤ 0, `pageSize` ≤ 0, `pageSize` > 100, or either value is non-integer **Then** I receive `400` RFC 7807 with a field-level problem detail identifying the offending parameter; no cards are returned.

**Given** a board has ~200 cards **When** I apply metadata filters **Then** the API responds in < ~1.5s (p95 on SQL Server with the required indexes in place).

**Given** I am viewing the board's filter panel **When** I apply or change metadata filters **Then** the card list updates without a full page reload; loading, empty-result, and error states are displayed; the UI meets WCAG 2.2 AA.

## NFR / constraints

- **Pagination:** offset-based; `page` (1-indexed, default 1) + `pageSize` (default 20, max 100); response envelope `{ items, page, pageSize, total }` — consistent with the shared convention.
- **Performance:** response < ~1.5s for a ~200-card board. Required SQL indexes:
  - Composite index on `Cards(BoardId, Priority)` — covers priority filter; `BoardId` prefix used by all card queries.
  - Composite index on `Cards(BoardId, DueDate)` — covers due-date range filters.
  - Join table `CardLabels(CardId, LabelId)` — index on `(LabelId, CardId)` to efficiently locate cards with a given label.
  - Join table `CardAssignees(CardId, UserId)` — index on `(UserId, CardId)`.
  - All indexes should be evaluated with `INCLUDE` columns for the projected DTO fields to support index-only or minimal heap lookups.
- **Authorization:** board membership verified server-side for every request; `boardId` in the route is the only scope (cross-board leakage is impossible by design). Roles Owner/Editor/Viewer may all read; no filter endpoint is write-scoped.
- **Filter semantics:** AND across filter types; OR within multi-value params (`labelIds`, `assigneeIds`). Semantics must appear in OpenAPI `description` fields.
- **Validation:** unknown `labelId`/`assigneeId` (not belonging to this board) returns `400`, not silent omission. Invalid date format returns `400`. Invalid `priority` enum value returns `400` with a field-level problem detail. `page` ≤ 0, `pageSize` ≤ 0 or > 100, or non-integer pagination params return `400`.
- **DTOs only** at the API boundary; input validated via `FluentValidation` or data annotations; RFC 7807 on every error path.
- **Clean Architecture:** query handler in Application layer; EF Core LINQ query in Infrastructure (no raw SQL unless a performance profiling spike justifies it, and that decision is documented); thin controller in Api.
- **No mutations:** this endpoint is strictly read-only.

## UX spec (skeleton)

Filter panel (collapsible sidebar or toolbar row above the board's card list):

- **Label filter:** multi-select chip/dropdown; labels are board-specific; shows label colour and name.
- **Assignee filter:** multi-select avatar/name picker; members of the board only.
- **Priority filter:** single-select segmented control (`None / Low / Medium / High`); one value may be active at a time; selecting an already-active value clears the filter (deselects).
- **Due-date range:** two date inputs (`Due after` / `Due before`); defaults empty; validates that after ≤ before.
- **Apply / Reset:** explicit Apply button (or auto-apply with debounce); Reset clears all metadata filters only.

States:
- **Loading:** spinner or skeleton rows in the card list while the filtered request is in flight.
- **Empty result:** illustration + copy "No cards match your filters" with a "Clear filters" link.
- **Error:** banner (`role="alert"`) with retry option.

Accessibility: all filter controls keyboard-navigable; `aria-label` on multi-selects; date inputs use `<input type="date">`; filter-result count announced via `aria-live="polite"` region.

## Out of scope

- Text / keyword search (STORY-8.2).
- Combining filters with text search in a single request (STORY-8.3).
- Active filter chip display and clear-individual-filter UX (STORY-8.3).
- Sorting the filtered result set (future).
- Cross-board filtering (EPIC-9).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `FilterCardsHandlerTests` (Application unit — all-filters match, partial-filter match, OR-within-label semantics, unknown-label → 400, invalid-date-range → 400, invalid-priority-enum → 400 field-level problem detail, page ≤ 0 → 400, pageSize ≤ 0 → 400, pageSize > 100 → 400, non-integer page/pageSize → 400, authz-denied → 404, empty-result → 200 empty envelope, pagination shape); `FilterCardsEndpointTests` (Api integration — 200 paginated, 400 bad-date RFC 7807, 400 unknown-label-id RFC 7807, 400 invalid-priority RFC 7807, 400 invalid-pagination-params RFC 7807, 401 unauthenticated, 404 non-member, cross-board label-id rejected, index-supported query plan assertion against a 200-card seed); `FilterPanel.spec.ts` (Vue/Vitest — renders label/assignee/single-select-priority/date controls, priority control is single-select (selecting a value deselects any prior value), apply triggers request, loading/empty/error states, keyboard nav, aria-live region, WCAG aa)
