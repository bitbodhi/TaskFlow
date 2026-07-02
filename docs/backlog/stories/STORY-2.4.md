# STORY-2.4: Reorder lists on a board

**Epic:** EPIC-2

As an Editor or Owner of a board, I want to drag and drop lists into a new order, so that I can arrange columns to reflect my workflow without losing any other data.

## Acceptance Criteria

**Given** I am an Owner or Editor of a board **When** I drag a list to a new position and drop it **Then** the list's rank is updated server-side, the board's visible column order reflects the new position immediately, and I receive `200 OK` with the moved list's updated DTO (id, boardId, name, position).

**Given** I drag a list to between two adjacent lists **When** the reorder is persisted **Then** only the moved list's `position` column changes — no other rows are rewritten — satisfying the concurrency-safe, minimal-write requirement.

**Given** I drag a list to the first position **When** the reorder is persisted **Then** the moved list's rank is lower than all others on the board.

**Given** I drag a list to the last position **When** the reorder is persisted **Then** the moved list's rank is higher than all others on the board.

**Given** I am an unauthenticated caller **When** I attempt to reorder a list **Then** I receive `401` and no list is moved.

**Given** I am authenticated but not a member of the board **When** I attempt to reorder a list **Then** I receive `404` (existence not disclosed to non-members) and no list is moved.

**Given** I am a Viewer of the board **When** I attempt to reorder a list **Then** I receive `403` (member whose role forbids the mutation) and no list is moved.

**Given** two users reorder different lists concurrently **When** both writes land **Then** both are accepted (no error) and the resulting order is consistent — each list's final rank reflects both moves without a lost-update anomaly.

**Given** the gap between two neighbouring ranks falls below `1e-9` **When** a reorder would need to insert between them **Then** the Application layer re-normalises all list ranks for the board (integer 1, 2, 3 …) in the same transaction before writing the target rank, so no error surfaces to the caller.

**Given** a reorder succeeds **When** another board member views the board **Then** real-time propagation < 1 s (via EPIC-6) updates the column order for connected clients.

**Given** the drag begins **When** the user releases outside a valid drop target (or presses `Esc`) **Then** the list snaps back to its original position and no server call is made.

**Given** an `afterListId` that belongs to a different board or does not exist **When** I PATCH the position **Then** I receive `404` RFC 7807 (existence not disclosed) and the position is unchanged.

## NFR / constraints

- **Position model (canonical definition — shared with STORY-2.1):** each list carries a `double`-precision `Position` column (fractional index). Insertion between predecessor `p` and successor `s` writes `(p + s) / 2`. Appending writes `max + 1.0`. Prepending writes `min - 1.0`. When the insertion gap < `1e-9`, the Application layer re-normalises all positions for the board to integers in the same transaction. This model ensures at most **one row written per reorder** under normal conditions.
- **Concurrency:** `PATCH /boards/{boardId}/lists/{listId}/position` is a single-row update (optimistic; no row-level locking needed for position writes in SQL Server with default READ COMMITTED). Concurrent reorders of *different* lists are serialisable without conflict. Concurrent reorders of the *same* list: last-write-wins (acceptable for v1 drag-and-drop).
- **API shape:** `PATCH /boards/{boardId}/lists/{listId}/position` with body `{ "afterListId": "<guid | null>" }` — `null` means move to first position. Returns `200 OK` with the updated list DTO.
- **Security / authorization:** reorder is permitted for Owner and Editor only. Status-code convention (per EPIC-1 / STORY-1.5): `401` unauthenticated; `404` non-member, list not found, or `afterListId` not found / belongs to a different board (existence not disclosed — all merged into one 404 with a generic message); `403` Viewer; `400` validation.
- **Keyboard / accessibility:** drag-and-drop is pointer-driven; a keyboard-accessible reorder mechanism is required for WCAG 2.2 AA (e.g. list options menu items "Move left" / "Move right", or arrow-key handling while the item has an `aria-grabbed` role).
- **Clean Architecture (§2.2):** reorder use-case in Application; Domain entity validates position bounds; EF Core single-row update in Infrastructure; thin controller in Api.
- **Real-time:** propagation constraint noted; EPIC-6 owns the SignalR story.

## UX spec (skeleton)

Board view renders lists as draggable columns (using a DnD library, e.g. vue-draggable-plus or dnd-kit equivalent for Vue). While dragging: the dragged column has reduced opacity and a placeholder gap shows its destination. Dropping fires `PATCH …/position`; during the API call the column stays at the new visual position (optimistic). On success: position confirmed silently. On error: column snaps back and an `role="alert"` error toast appears. **Keyboard alternative:** each list's options menu (`⋮`) contains "Move left" and "Move right" items (disabled at the respective boundary); activating one sends the same PATCH. WCAG 2.2 AA: drag handle has `aria-roledescription="sortable"` and `aria-grabbed`; keyboard alternative meets SC 2.1.1. No external comp — this note is the design reference.

## Out of scope

- Card reorder / DnD (EPIC-3).
- Multi-list selection and batch reorder.
- Animated transitions beyond the drag placeholder.
- Real-time delivery story (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.5, §1.6, §2.2
**Tests:** `ReorderListHandlerTests` (Application unit — move to first, move to last, move between two lists, re-normalisation triggered when gap < 1e-9, concurrent-same-list last-write-wins does not error, authz denied Viewer, authz denied non-member, afterListId on a different board → 404 generic, afterListId not found → 404 generic); `ReorderListEndpointTests` (Api integration — 200 DTO with updated position, 401, 403 Viewer, 404 non-member, 404 wrong-list-id, 404 afterListId-wrong-board, 404 afterListId-not-found, verify ordering via subsequent GET); `BoardListsDnd.spec.ts` (Vue/Vitest — drop fires PATCH, optimistic update, snap-back on error, keyboard move-left/move-right menu items, a11y attributes)
