# EPIC-4: Card metadata — labels, assignees, due dates, priorities

**Goal:** Editors and Owners can enrich cards with board-scoped labels, member assignees, due dates, and priorities, so that work can be categorised, owned, and tracked across a board.
**FRs covered:** Labels & filtering (§1.3, §1.5); assignees (§1.3, §1.4); due dates and priorities (§1.3). Assignee pool is the board's member list (§1.4). Card entity established by EPIC-3; member roster established by EPIC-1.
**Traces:** §1.3, §1.4, §1.6, §2.2

## Stories
- STORY-4.1 — Manage a board's label set (create / edit / delete)
- STORY-4.2 — Add / remove a label on a card
- STORY-4.3 — Assign / unassign a board member to a card
- STORY-4.4 — Set / clear a card due date
- STORY-4.5 — Set / clear a card priority

## Out of scope
- Filtering / searching cards by label, assignee, due date, or priority (EPIC-8).
- "My Tasks" aggregated view across boards (EPIC-9).
- Real-time push of metadata changes to other connected clients (EPIC-6 authors all SignalR stories; stories here note the propagation expectation only).
- File attachments, checklists, custom fields.
- Label colour as the sole distinguisher — labels always carry a required text name (WCAG 2.2 AA).

## Notes
- **Label cascade:** deleting a board label removes all card–label associations for that label; no orphaned associations.
- **Assignee guard:** a user who has been removed from the board cannot be newly assigned; existing assignments are preserved until explicitly removed (removal is surfaced as a warning in the UI).
- **Timezone:** due dates stored as UTC `DateTimeOffset`; overdue indication is a display/client concern.
- **Priority enum (canonical):** `None` / `Low` / `Medium` / `High` — defined in Domain; stored as `int` column; surfaced as string in API DTOs.
- **Pagination envelope** (all list endpoints): offset-based; `page` (1-indexed, default 1), `pageSize` (default 20, max 100); `{ items, page, pageSize, total }`.
- **Status codes** (shared convention): `401` unauthenticated; `404` non-member (existence not disclosed); `403` role forbids action (Viewer mutating); `400` validation error. RFC 7807 on every error path.
