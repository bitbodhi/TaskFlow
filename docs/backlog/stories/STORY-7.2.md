# STORY-7.2: View a board's activity feed

**Epic:** EPIC-7

As a board member (Owner, Editor, or Viewer), I want to browse a reverse-chronological log of everything that has changed on the board, so that I can understand what happened and who did it without asking teammates.

## Acceptance Criteria

**Given** I am a board member (any role) **When** I `GET /boards/{boardId}/activity` **Then** I receive `200 OK` with a paginated envelope `{ items, page, pageSize, total }` where `items` is the list of activity entries sorted newest-first, each entry containing `id`, `eventType`, `description`, `actorId`, `actorDisplayName`, `occurredAt` (ISO 8601 UTC). Default `page=1`, `pageSize=20`; max `pageSize=100`.

**Given** I am not a member of the board, or present no/invalid JWT **When** I `GET /boards/{boardId}/activity` **Then** I receive `404` RFC 7807 (non-member — existence not disclosed) or `401` (unauthenticated) respectively. No data is returned.

**Given** the board has no activity entries yet **When** I `GET /boards/{boardId}/activity` **Then** I receive `200 OK` with `{ items: [], page: 1, pageSize: 20, total: 0 }`.

**Given** `pageSize` exceeds 100 or `page` is less than 1 **When** I call the endpoint **Then** I receive `400` RFC 7807 with a descriptive problem detail.

**Given** I open the board's Activity tab in the UI **When** the page loads **Then** the feed renders in reverse-chronological order: each entry shows the actor's display name (or "System"), the human-readable `description`, and the relative or absolute `occurredAt` timestamp. A loading skeleton is shown during fetch. An empty state ("No activity yet") is shown if the board has no entries. An error state with a retry button is shown on fetch failure.

**Given** there are more entries than fit on one page **When** I reach the bottom of the feed **Then** the UI loads the next page (infinite scroll or a "Load more" button), appending older entries below.

**Given** I open the Activity tab on a board I am a Viewer of **When** the page loads **Then** the feed renders exactly as for an Owner or Editor (read-only is the same view; Viewer has no additional restrictions on reading the feed).

**Given** the UI renders an entry for a deleted target (e.g. a card that was later deleted) **When** I view that entry **Then** the description snapshot (captured at write time) is displayed intact; no broken reference or missing-data placeholder.

## NFR / constraints
- **Authorization:** enforced server-side on every request. Role check: any board member (Owner, Editor, Viewer) may read. Non-member → `404`; unauthenticated → `401`. Status code convention matches §1.6 and STORY-0.3/0.4 shared convention.
- **Pagination:** offset-based; same envelope as all other list endpoints: `{ items, page, pageSize, total }`, `page` 1-indexed default 1, `pageSize` default 20 max 100. The endpoint MUST always paginate — no unbounded list responses.
- **Read-only:** no mutation (`POST`/`PUT`/`PATCH`/`DELETE`) is exposed on `/activity`. This endpoint is GET only.
- **Performance:** the `(BoardId, OccurredAt DESC)` index (created in STORY-7.1) makes page reads efficient. The endpoint must not load all entries into memory before paginating.
- **DTOs only at the boundary (§2.2):** the API returns an `ActivityEntryDto` (id, eventType, description, actorId, actorDisplayName, occurredAt). The `ActivityEntry` domain entity is never serialised directly.
- **Clean Architecture (§2.2):** query handler in Application; EF Core read in Infrastructure; thin controller in Api; no EF/Microsoft.* in Domain.
- **RFC 7807:** all error paths (`400`, `401`, `404`) return a valid problem detail.
- **Accessibility (§1.6 WCAG 2.2 AA):** feed list is a semantic `<ol>` or `<ul>`, timestamps use `<time datetime="…">`, loading skeleton is `aria-busy`, empty/error states are announced via `role="status"` or `role="alert"`. Infinite scroll / Load-more is keyboard-accessible.
- **Timestamps:** `occurredAt` is returned in ISO 8601 UTC from the API. The UI may render it as a relative time (e.g. "3 minutes ago") with the full absolute timestamp in a `title` attribute or tooltip.

## UX spec (skeleton)

**Location:** A dedicated **Activity** tab on the board detail page (alongside the lists view).

**Feed list:** Reverse-chronological `<ol>`. Each item:
- Actor avatar (initials fallback) + bold display name
- Description string (e.g. "moved card 'Fix login bug' to 'In Progress'")
- Relative timestamp (e.g. "3 min ago") with full UTC absolute in `title` / `aria-label`

**States:**
- **Loading:** skeleton rows (`aria-busy="true"` on the list container) while the first page fetches.
- **Empty:** centred illustration/icon + "No activity on this board yet." (`role="status"`).
- **Error:** banner with error message and "Retry" button (`role="alert"`).
- **Paginated load-more:** a "Load more" button below the list (or `IntersectionObserver` infinite scroll). While loading additional pages, a spinner appears below the current entries.

**Keyboard / a11y:**
- Tab to "Load more" button; Enter/Space triggers it.
- Each feed item is a list item — no interactive elements required (read-only).
- Timestamps: `<time datetime="2026-06-30T14:22:00Z">3 min ago</time>`.

No external Figma comp — this skeleton is the design reference.

## Out of scope
- Filtering or searching within the feed (EPIC-8).
- Live-updating the feed when new entries arrive (STORY-7.3).
- Editing or deleting entries from the UI (immutable by design).
- Exporting the activity log.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.5, §1.6, §2.2
**Tests:** `GetBoardActivityQueryHandlerTests` (Application unit — correct page shape, empty board returns empty page, pageSize>100 rejected, page<1 rejected); `GetBoardActivityEndpointTests` (Api integration — 200 paginated DTO shape, 401 unauthenticated, 404 non-member, 400 invalid pagination params, Viewer role can read, entries sorted newest-first; POST/PUT/PATCH/DELETE on `/boards/{boardId}/activity` return `405 Method Not Allowed`); `ActivityFeed.spec.ts` (Vue/Vitest — loading skeleton rendered during fetch, empty state, error state with retry, "Load more" appends next page, each entry shows actor name + description + formatted timestamp, a11y attributes present).
