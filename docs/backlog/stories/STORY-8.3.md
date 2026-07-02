# STORY-8.3: Combine filters and text search, display active filter chips, and clear them

**Epic:** EPIC-8

As a board member, I want to apply both text search and metadata filters simultaneously, see which filters are active as dismissible chips, and clear any or all of them with one action, so that I can iteratively narrow my results and always understand what is shaping the card list.

## Acceptance Criteria

**Given** I am a member of a board **When** I call `GET /boards/{boardId}/cards` with any combination of `q`, `labelIds`, `assigneeIds`, `priority`, `dueBefore`, and `dueAfter` supplied together **Then** I receive `200` with a paginated envelope `{ items, page, pageSize, total }` containing only cards that satisfy ALL supplied predicates simultaneously (text match AND all metadata filters); omitted params are not applied.

**Given** I supply a combination that individually produces results but their intersection is empty **When** I call the endpoint **Then** I receive `200` with `{ items: [], page: 1, pageSize: <requested>, total: 0 }` — an empty intersection is not an error.

**Given** I supply any invalid parameter value (bad date, `q` > 200 chars, unknown `labelId`/`assigneeId`) **When** I call the endpoint **Then** I receive `400` RFC 7807 with field-level problem details covering all validation failures in a single response (do not short-circuit on the first error).

**Given** I supply a `page` or `pageSize` param **When** `page` ≤ 0, `pageSize` ≤ 0, `pageSize` > 100, or either value is non-integer **Then** I receive `400` RFC 7807 with a field-level problem detail identifying the offending parameter; no cards are returned.

**Given** I am not a member of the board (or the board does not exist) **When** I call the combined endpoint **Then** I receive `404` RFC 7807; the board's existence is not disclosed.

**Given** I present no token or an expired token **When** I call the endpoint **Then** I receive `401` RFC 7807.

**Given** a board has ~200 cards **When** I apply a combined query (text + two metadata filters) **Then** the API responds in < ~1.5s (p95), reusing the indexes established in STORY-8.1 and STORY-8.2.

**Given** I have applied one or more filters or a search term **When** I view the board **Then** each active predicate is shown as a labelled chip (e.g. "Label: Bug", "Assignee: Alice", "Priority: High", "Due before: 2026-07-31", `q`: "payment"); chips are keyboard-focusable and dismissible individually without navigating away. The `priority` filter is single-select (at most one priority chip is ever shown); dismissing the priority chip clears the single selected value and the `priority` query param is omitted from the next request.

**Given** I click or keyboard-activate the × on an individual chip **When** the chip closes **Then** that single predicate is removed, the remaining active filters are preserved, and the card list refreshes with the updated (narrower) filter set.

**Given** one or more filters are active **When** I click or keyboard-activate "Clear all filters" **Then** all predicates (text + metadata) are reset to their defaults (absent), the chip row disappears, the card list returns to the full unfiltered board view, and the search input and all filter controls are reset to their empty/default states.

**Given** I clear all filters **When** no filters remain active **Then** the chip row and "Clear all filters" control are hidden (not rendered as an empty row).

**Given** the board has no cards at all **When** I open it (no filters active) **Then** the standard board-level empty state is shown, not the "no filter results" empty state.

**Given** all filters are valid but produce no cards **When** the empty-result state is rendered **Then** it includes a "Clear all filters" affordance so the user can recover without manually undoing each chip.

**Given** the combined search/filter UI **When** I interact with it **Then** all controls and chip interactions meet WCAG 2.2 AA (keyboard nav, focus management after chip removal, screen-reader announcements).

## NFR / constraints

- **Single endpoint:** `GET /boards/{boardId}/cards` accepts all params from STORY-8.1 and STORY-8.2 in one call; the Application-layer query handler composes predicates dynamically — each absent param is a no-op. No separate "combined" endpoint; STORY-8.1 and STORY-8.2 already implement the same route; STORY-8.3 adds the multi-predicate composition and the frontend UX layer.
- **Validation:** collect all field-level validation errors before returning `400`; do not halt on the first failure. Use `FluentValidation` `RuleFor` chaining or equivalent to surface all problems in one RFC 7807 `errors` map. Invalid pagination params (`page` ≤ 0, `pageSize` ≤ 0 or > 100, non-integer) are included in this collected validation.
- **Pagination:** offset-based; `page` (1-indexed, default 1) + `pageSize` (default 20, max 100); envelope `{ items, page, pageSize, total }`. Pagination resets to page 1 on every filter change (frontend responsibility).
- **Performance:** < ~1.5s p95 for ~200-card boards. The indexes from STORY-8.1 (`Cards(BoardId, Priority)`, `Cards(BoardId, DueDate)`, `CardLabels(LabelId, CardId)`, `CardAssignees(UserId, CardId)`) and STORY-8.2 (`Cards(BoardId) INCLUDE (Title, Description)`) together cover all predicate combinations. EF Core LINQ composes predicates as a single SQL query; no N+1.
- **Authorization:** board membership verified server-side; all predicates evaluated within the board's row set. Read-only; no role distinction among Owner/Editor/Viewer for reading.
- **Client state:** active filters and search term are held in a Pinia store slice (or a composable) so chip state and filter-panel state remain in sync without prop-drilling. URL query-string sync is optional (future nice-to-have; not an AC here).
- **Focus management:** after removing a chip via keyboard, focus moves to the next chip if one exists, otherwise to the "Clear all filters" button, otherwise to the search input. Announced via `aria-live="assertive"` for chip removal.
- **DTOs only** at the boundary; RFC 7807 on every error path.
- **Clean Architecture:** the combined query handler in Application composes EF Core predicates; Infrastructure layer executes one SQL query; thin controller in Api.

## UX spec (skeleton)

The board view has a unified search/filter toolbar above the card list. From STORY-8.1 and STORY-8.2, the toolbar already contains a search input and a filter panel. STORY-8.3 adds the active-filter chip strip between the toolbar and the card list:

**Active filter chip strip** (rendered only when at least one predicate is active):

```
[× Label: Bug] [× Assignee: Alice] [× Priority: High] [× q: "payment"]   [Clear all filters ×]
```

- Each chip: labelled with the filter type and value, `role="group"` or a button with `aria-label="Remove filter: Label Bug"`.
- "Clear all filters" button at the end of the strip; hidden when strip is empty.
- Strip disappears (not collapsed to an empty row) when all chips are cleared.

**Combined interaction flow:**

1. User types in the search input (debounced 300 ms) → `q` chip appears; card list refreshes.
2. User opens filter panel, selects label "Bug" and assignee "Alice" → chips appear; card list refreshes with intersection.
3. User clicks × on "Assignee: Alice" chip → chip removed, list refreshes with `q` + "Label: Bug" only.
4. User clicks "Clear all filters" → all chips removed, search input cleared, filter panel reset, full card list shown.

**Empty-result state** (all filters valid, zero cards returned):

```
[Illustration]
No cards match your current filters.
[Clear all filters]
```

The "Clear all filters" link in the empty state is the same action as the toolbar button.

States: loading (spinner/skeleton on card list), results (chip strip + filtered list), empty (empty-result UX), error (`role="alert"` banner + retry).

Accessibility: chip buttons have descriptive `aria-label`; strip container has `aria-label="Active filters"`; focus management on chip removal (see NFR); result count update via `aria-live="polite"`; "Clear all filters" is keyboard-reachable; all colour usage has sufficient contrast.

## Out of scope

- Cross-board search (EPIC-9).
- Saving, naming, or sharing filter presets (future).
- URL query-string sync / deep-linking to a filtered view (future).
- Sorting or reordering the filtered result set (future).
- Searching comments, activity, or attachments.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `CombinedSearchFilterHandlerTests` (Application unit — text+label match, text+label+assignee intersection, empty-intersection → 200 empty, multi-error 400 collects all failures including invalid pagination params, missing-q no-op, missing-metadata no-op, page ≤ 0 → 400, pageSize > 100 → 400, authz-denied → 404, pagination resets); `CombinedSearchFilterEndpointTests` (Api integration — 200 correct intersection with 200-card seed, 400 collects all invalid params in one response including bad pagination, 401, 404 non-member, no cross-board leakage, single SQL query issued per request); `ActiveFilterChips.spec.ts` (Vue/Vitest — chip rendered per active predicate, priority yields at most one chip (single-select), dismissing priority chip clears single value and omits priority param from next request, individual chip × removes only that predicate and triggers refresh, clear-all resets all controls, chip strip hidden when no active filters, empty-state "Clear all filters" link, focus management after chip removal, aria-live result count, WCAG aa)
