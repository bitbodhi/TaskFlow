# STORY-8.2: Text-search a board's cards by title and description

**Epic:** EPIC-8

As a board member, I want to type a keyword and have the board surface cards whose title or description contains that text, so that I can locate a specific card instantly without knowing which list it is in.

## Acceptance Criteria

**Given** I am a member of a board **When** I call `GET /boards/{boardId}/cards?q=<term>` with a non-empty query string **Then** I receive `200` with a paginated envelope `{ items, page, pageSize, total }` containing only cards whose `title` or `description` contains the term (case-insensitive, substring/LIKE match); cards are scoped to this board only.

**Given** the query matches multiple fields on the same card **When** the card is returned **Then** it appears exactly once in the result set (no duplicates from multi-field matching).

**Given** I supply `q` with leading or trailing whitespace **When** the query is evaluated **Then** the whitespace is trimmed server-side and the trimmed term is used; an all-whitespace `q` is treated as absent (see the blank `q` criterion below).

**Given** I omit `q` or supply a blank/whitespace-only `q` **When** I call the search endpoint **Then** the text-search predicate is not applied and all cards on the board are returned (or the intersection with any other supplied filters if STORY-8.3 is combined); this is not an error.

**Given** I supply a `q` term longer than 200 characters **When** I call the endpoint **Then** I receive `400` RFC 7807 with a clear problem detail; no query is executed.

**Given** I supply a `page` or `pageSize` param **When** `page` ≤ 0, `pageSize` ≤ 0, `pageSize` > 100, or either value is non-integer **Then** I receive `400` RFC 7807 with a field-level problem detail identifying the offending parameter; no cards are returned.

**Given** I supply a `q` containing SQL LIKE metacharacters (`%`, `_`, `[`) **When** the query is evaluated **Then** those characters are treated as literal text — they do not function as wildcards; a `q` of `"100%"` must match only cards whose title or description contains the literal string `100%`, not all cards.

**Given** I am not a member of the board (or the board does not exist) **When** I call the search endpoint **Then** I receive `404` RFC 7807; the board's existence is not disclosed.

**Given** I present no token or an expired token **When** I call the search endpoint **Then** I receive `401` RFC 7807.

**Given** the search term matches no cards **When** I call the search endpoint **Then** I receive `200` with `{ items: [], page: 1, pageSize: <requested>, total: 0 }`.

**Given** a board has ~200 cards **When** I search with a term that hits a significant portion of cards **Then** the API responds in < ~1.5s (p95 with the required index in place).

**Given** I am viewing the board **When** I type into the search input **Then** the card list filters live (debounced, ≥ 300 ms) without a full page reload; loading, empty-result, and error states are shown; the UI meets WCAG 2.2 AA.

## NFR / constraints

- **Matching strategy:** case-insensitive **LIKE '%term%'** (substring) against `Cards.Title` and `Cards.Description`. This is the v1 choice — simple, zero-migration, sufficient for a ~200-card board. A migration to SQL Server Full-Text Search (FTS) is the documented upgrade path when corpus size or relevance ranking demands it, but is out of scope here.
- **Performance:** response < ~1.5s for a ~200-card board. Required index:
  - Index on `Cards(BoardId)` covering `Title` and `Description` columns (or an existing `BoardId` index with `INCLUDE (Title, Description)`) — limits the LIKE scan to only the target board's rows, avoiding a full-table scan.
  - Note: LIKE with a leading wildcard (`'%term%'`) cannot use a B-tree range scan on the text columns; the `BoardId` prefix reduces the scanned set to at most the board's own cards (~200 rows in the target size), which is acceptable. If corpus grows, adding an FTS catalog on `(Title, Description)` is the upgrade path.
- **Pagination:** offset-based; `page` (1-indexed, default 1) + `pageSize` (default 20, max 100); envelope `{ items, page, pageSize, total }`.
- **Validation:** `q` > 200 characters → `400` RFC 7807. Blank/whitespace `q` → treated as absent (not `400`). `page` ≤ 0, `pageSize` ≤ 0 or > 100, or non-integer pagination params → `400` RFC 7807 field-level problem detail.
- **LIKE metacharacter escaping:** `%`, `_`, and `[` in `q` must be escaped before being passed to the LIKE predicate (EF Core `.Contains()` handles this automatically; if parameterised SQL is used instead, the caller must escape them manually). A `q` containing these characters must never expand to an unintended wildcard match.
- **Authorization:** board membership verified server-side; result set never leaks cards from other boards. Roles Owner/Editor/Viewer may all search; endpoint is read-only.
- **DTOs only** at the boundary; RFC 7807 on every error path.
- **Clean Architecture:** search query handler in Application; EF Core LINQ (`.Contains()` translates to `LIKE '%…%'`) or parameterised SQL in Infrastructure; thin controller in Api.
- **No mutations:** strictly read-only.
- **Debounce:** frontend fires the request no more than once per 300 ms of typing inactivity to avoid excess API calls; in-flight requests for a superseded term are cancelled (AbortController).

## UX spec (skeleton)

Search input bar at the top of the board view (persistent, not modal):

- Single `<input type="search">` with a magnifying-glass icon button and a clear (×) button (appears when the field is non-empty).
- Placeholder: "Search cards…".
- Debounced live search (300 ms idle). While a request is in flight the card list shows a skeleton/spinner overlay; the input remains editable.
- **Matched text highlighting:** bold/highlighted occurrences of the search term within the card title in the result list (client-side highlight only; no server-side markup required).
- **Result count:** "N cards found" updated via `aria-live="polite"` region after each response.

States:
- **Idle (no query):** full card list shown (all lists, all cards as paginated).
- **Loading:** spinner/skeleton overlay on the card list; ARIA busy state on the list container.
- **Results:** filtered list with term highlighted.
- **Empty result:** "No cards match '<term>'" with a "Clear search" link.
- **Error:** `role="alert"` banner with retry.

Accessibility: search input has `aria-label="Search cards"`; clear button has `aria-label="Clear search"`; result count region uses `aria-live="polite"`; keyboard: Tab into input, Enter or idle-debounce fires search, Esc clears the field and restores full list.

## Out of scope

- Combining text search with metadata filters in a single request (STORY-8.3).
- Relevance ranking or fuzzy matching (future / FTS upgrade path).
- Searching in comments, activity log, or card attachments.
- Cross-board search (EPIC-9).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `SearchCardsHandlerTests` (Application unit — single-field title match, single-field description match, case-insensitive, trim whitespace, blank q → all cards, q > 200 chars → 400, no match → 200 empty, authz-denied → 404, pagination shape, page ≤ 0 → 400, pageSize ≤ 0 → 400, pageSize > 100 → 400, non-integer page/pageSize → 400, q containing `%` matches literal `%` only, q containing `_` matches literal `_` only, q containing `[` matches literal `[` only); `SearchCardsEndpointTests` (Api integration — 200 paginated result set with correct item set, 400 q-too-long RFC 7807, 400 invalid-pagination-params RFC 7807, 400 q-with-metacharacters-does-not-wildcard-expand (seed board with card titled `100%done` and another with any title; q=`100%` returns only the literal-match card), 401, 404 non-member, no-cross-board leakage with a 200-card seed, query plan uses BoardId index); `SearchBar.spec.ts` (Vue/Vitest — debounce fires after 300 ms idle, AbortController cancels superseded request, loading/empty/error states, clear button, aria-live count, keyboard Esc clears, WCAG aa)
