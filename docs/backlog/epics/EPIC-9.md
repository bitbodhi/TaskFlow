# EPIC-9: My Tasks — cross-board assigned card aggregation

**Goal:** A user sees, in one place, every card assigned to them across all boards they are a member of, can triage those cards by sorting and filtering, and can navigate directly to any card in its board — without ever surfacing a card from a board they cannot access.
**FRs covered:** "My Tasks" cross-board view (§1.3, §1.5); per-board roles and membership boundary (§1.4); assignees established by EPIC-4; board membership established by EPIC-1. Aggregation must respect the membership boundary at query time.
**Traces:** §1.4, §1.5, §1.6, §2.2

## Stories
- STORY-9.1 — View my assigned cards across all my boards (paginated, with isolation guarantee)
- STORY-9.2 — Sort and filter My Tasks for triage
- STORY-9.3 — Navigate from a My Tasks item to the card in its board (deep link)

## Out of scope
- Assigning or unassigning cards from the My Tasks view (mutation belongs to card detail, EPIC-4).
- Filtering by label within My Tasks (EPIC-8 owns within-board filters; label filter on My Tasks is a future cross-epic concern).
- Real-time push of assignment changes into the My Tasks list (EPIC-6 owns SignalR wiring; stories here note the propagation expectation only — the view refreshes on next load).
- Notification or email delivery for new assignments.
- Bulk actions (mark done, reassign) on multiple cards at once.

## Notes
- **Isolation is the invariant.** The query joining assignees → cards → lists → boards MUST filter by the current user's active board membership at query time. A card dropped from an assigned board (STORY-1.4 removes the user) must disappear from the next My Tasks fetch — no stale results.
- **Index requirement.** At scale a user may be a member of many boards each with many cards. The assignee → card look-up must be backed by a database index on `(UserId, CardId)` in the assignee join table (or equivalent) to satisfy the <1.5 s board-load NFR (§1.6).
- **Pagination envelope** (all list endpoints): offset-based; `page` (1-indexed, default 1), `pageSize` (default 20, max 100); `{ items, page, pageSize, total }`.
- **Status codes** (shared convention): `401` unauthenticated; `400` invalid query params; `404` is not used for My Tasks (it is the caller's own data — nothing to hide). The critical negative guarantee: `200` with zero items when the user has no assignments, never a `404`.
- **Due-date display:** stored as UTC `DateTimeOffset`; overdue indication is a client-side display concern (compare against the browser's local "now").
- EPIC-1 (membership) and EPIC-4 (assignees) are prerequisites. EPIC-8 (within-board filter) is a separate concern and is not a prerequisite.
