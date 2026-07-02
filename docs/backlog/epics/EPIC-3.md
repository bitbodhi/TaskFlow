# EPIC-3: Cards — Full Lifecycle, Move & Reorder (Drag-and-Drop)

**Goal:** Editors and Owners can open, edit, move, reorder, and delete cards — including drag-and-drop reordering within a list and moving cards across lists on the same board — with a stable, concurrency-safe ordering model. Card metadata (labels, assignees, due dates, priorities) is EPIC-4; comments are EPIC-5; real-time broadcast of these mutations is EPIC-6.

**FRs covered:** Cards CRUD with ordering; drag-and-drop positioning (§1.3, §1.5); per-board role enforcement — Editor+ mutates, Viewer read-only (§1.4); performance NFR board ≤1.5s with ~200 cards (§1.6).

**Traces:** §1.1, §1.3, §1.4, §1.5, §1.6, §2.2

## Stories
- STORY-3.1 — View card details
- STORY-3.2 — Edit a card (title and description)
- STORY-3.3 — Delete a card
- STORY-3.4 — Move a card to another list on the same board
- STORY-3.5 — Reorder a card within a list (drag-and-drop)

## Out of scope
- Card metadata: labels, assignees, due dates, priorities (EPIC-4).
- Comments (EPIC-5).
- Real-time propagation of mutations to other connected clients (EPIC-6 — referenced as a constraint in each story).
- Moving a card to a list on a different board (rejected as a validation error).
- Ordering model changes caused by concurrent reorders on the same list from different clients are resolved deterministically by the server; conflict UI is out of scope.

## Notes
- **Ordering model:** cards carry a `rank` field of type `float` (double precision). On create, new cards append at `max(rank) + 1.0`. On reorder, the new rank is the midpoint of the two neighbours' ranks; if the gap falls below a configurable epsilon (e.g. `< 0.0001`) the server triggers a background renumbering of the list (assigns `1.0, 2.0, 3.0, …`) in a single transaction. This avoids rewriting every row on every move while guaranteeing a stable order under concurrent moves. All list-of-cards endpoints return rows `ORDER BY rank ASC` and use the shared pagination envelope.
- The status code convention established in EPIC-0 / STORY-0.3 applies throughout: `401` unauthenticated; `404` non-member (existence not disclosed); `403` role forbids action (Viewer mutating); `400` validation failure. All errors return RFC 7807 problem details.
