# STORY-3.5: Reorder a card within a list (drag-and-drop)

**Epic:** EPIC-3

As an Editor or Owner of a board, I want to drag a card to a new position within the same list, so that I can control the priority order of tasks without moving them to a different workflow stage.

## Acceptance Criteria

**Given** I am an authenticated Editor or Owner of the board **When** I PATCH `/cards/{cardId}/reorder` with a valid `afterCardId` (the card after which to insert; `null` or omitted means place at the top) **Then** the card's `rank` is updated to place it immediately after `afterCardId` within the same list, I receive `200 OK` with the updated card DTO (`id`, `title`, `listId`, `rank`, `updatedAt`), and a subsequent paginated GET of the list's cards returns cards ordered by the new rank.

**Given** I supply an `afterCardId` that belongs to a **different list** than the card being reordered **When** I PATCH `/cards/{cardId}/reorder` **Then** I receive `400` RFC 7807 ("Position anchor does not belong to the same list") and the card's rank is unchanged.

**Given** I supply an `afterCardId` equal to the card being reordered **When** I PATCH `/cards/{cardId}/reorder` **Then** I receive `400` RFC 7807 ("A card cannot be positioned after itself") and no change is made.

**Given** `afterCardId` is null or omitted **When** I PATCH `/cards/{cardId}/reorder` **Then** the card is moved to rank position 0 (or midpoint between 0 and the current first card's rank), placing it at the top of the list.

**Given** two concurrent reorder operations attempt to place cards at the same midpoint position **When** both are processed **Then** the server resolves the tie deterministically (by `cardId` tiebreak within the gap-renumber pass if triggered, or by the natural race of the transactions); the resulting order is stable and consistent — no duplicate rank values exist in a list after either transaction commits.

**Given** I am a Viewer on the board **When** I PATCH `/cards/{cardId}/reorder` **Then** I receive `403` RFC 7807 and the card's rank is unchanged.

**Given** I am not a member of the board, or present no/invalid JWT **When** I PATCH `/cards/{cardId}/reorder` **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated) and no change is made.

**Given** I drag a card within a list column in the board UI **When** I drop it between two other cards **Then** the card appears at the new position immediately (optimistic reorder), the PATCH is issued in the background, and if the request fails the card snaps back to its original position with an error toast.

**Given** the board list has ~200 cards **When** a single reorder is performed **Then** the PATCH request completes in < ~500ms (well within the board's 1.5s load budget) and at most one additional renumber transaction is triggered (not N writes proportional to card count).

**Given** the board UI renders cards in a list column **When** any reorder completes **Then** the visual order of cards matches the `rank`-ordered server response — no local order drift.

**Given** a list column renders draggable cards **When** viewed by a Viewer **Then** cards are not draggable — no drag handle or drag affordance is shown.

**Given** the drag-and-drop UI is displayed **When** rendered **Then** it meets WCAG 2.2 AA: each card's drag handle is keyboard-accessible (arrow keys to move up/down, Space/Enter to pick up and drop), drag state is announced via `aria-grabbed`, valid drop targets indicated via `aria-dropeffect`, visible focus on the drag handle.

## NFR / constraints
- **Ordering model — fractional rank with gap strategy:**
  - Each card in a list has a `rank` column (`FLOAT` / `double` precision, non-null, default `0.0`).
  - On **create**: `rank = max(existing ranks in list) + 1.0`; first card in an empty list gets `rank = 1.0`.
  - On **reorder** (insert after card A, before card B): `new_rank = (A.rank + B.rank) / 2`. Edge cases: insert at top → `new_rank = first_card.rank / 2` (or `0.5` if the first card has rank `1.0`); insert at bottom → `new_rank = last_card.rank + 1.0`.
  - **Gap renumber trigger:** if `|new_rank - A.rank| < epsilon` (configurable, default `1e-9`) or `|B.rank - new_rank| < epsilon`, the server renumbers **all cards in that list** in a single transaction: assigns `1.0, 2.0, 3.0, …` in ascending rank order, then places the moved card at its correct position. This is O(n) writes but only triggered when the float space is exhausted, which takes thousands of moves on a single gap.
  - **Concurrency:** the reorder command reads the list's card ranks inside a serializable-isolation transaction (or a `SELECT … WITH (UPDLOCK)` hint on SQL Server) to prevent two concurrent midpoints from landing on the same value. The renumber pass similarly holds a lock on the list's rows. No ETag / optimistic-concurrency header required in v1.
  - **Query order:** all list-of-cards endpoints `ORDER BY rank ASC, id ASC` (id as tiebreaker; should not be needed after renumber but is a deterministic safety net).
- **Performance:** a single reorder must not rewrite all card rows except during the gap-renumber pass. Normal moves are a single-row UPDATE. The `rank` column is indexed on `(listId, rank)` to support the ORDER BY efficiently for the ~200-card board load.
- **Pagination:** the paginated GET of a list's cards (`{ items, page, pageSize, total }`) returns cards ordered by rank. Reordering does not break pagination (items are re-ranked before the next fetch, not in the client).
- **Real-time:** other board members will not see the reorder until EPIC-6 broadcasts a `card.reordered` event — noted as a constraint, not authored here. The constraint means optimistic UI is the only way a user sees their own reorder immediately.
- Authorization: Editor and Owner may reorder; Viewer receives `403`. Checked server-side.
- DTOs only at the API boundary; RFC 7807 on all error paths; structured logging of reorder events (cardId, fromRank, toRank, renumberTriggered).
- Clean Architecture: rank computation logic lives in the Application layer (command handler), not in Infrastructure or the Domain entity. The `IRankingService` interface (Application layer) is the seam; its SQL implementation is in Infrastructure.

## UX spec (skeleton)
Each card in a list column has a **drag handle** (grip icon, left edge of card, visible on hover/focus). Drag behaviour: pick up card → ghost follows cursor → list column shows a drop indicator line between cards → drop places card. On keyboard: focus drag handle → Space picks up → Up/Down arrows move card position → Space drops. Drag libraries (e.g. Vue Draggable, Sortable.js) are acceptable; must not break WCAG focus management. States: **idle** (cards listed in rank order), **dragging** (ghost + drop indicator), **submitting** (card at new position, reduced opacity, spinner on handle), **error** (card snaps back, toast `role="alert"` with error text), **success** (card rendered at new position, opacity restored). Viewer: drag handle not rendered; `draggable="false"` on card element. Empty list: drop zone still accepts dragged cards from STORY-3.4 (cross-list move). WCAG 2.2 AA: `aria-grabbed="true"` on picked-up card, `aria-dropeffect="move"` on valid drop targets, `aria-label` on drag handle ("Drag to reorder {cardTitle}").

## Out of scope
- Cross-list moves via drag-and-drop (that is STORY-3.4; the two share the drag mechanism but call different endpoints).
- Bulk reorder (reordering multiple selected cards at once — deferred).
- Custom sort modes (e.g. sort by due date — deferred to EPIC-4/EPIC-8).
- Persisting a user's personal view order separately from the shared board order (deferred).
- Undo / redo of reorder (deferred).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.3, §1.4, §1.5, §1.6, §2.2
**Tests:** `ReorderCardCommandHandlerTests` (Application unit — insert after card A success, insert at top null afterCardId, insert at bottom, afterCardId from different list → 400, afterCardId equals self → 400, rank midpoint computed correctly, gap-below-epsilon triggers renumber with correct final order, concurrent reorder tiebreak deterministic, Viewer → authz exception, non-member → not-found); `ReorderCardEndpointTests` (Api integration — 200 DTO with updated rank, paginated GET returns cards in new rank order, 400 cross-list anchor, 400 self-reference, 401 no token, 403 Viewer, 404 non-member, single-row UPDATE confirmed for non-renumber case via query count assertion); `CardListColumn.spec.ts` (Vitest — drag handle present for Editor, absent for Viewer, optimistic reorder on drag-drop, snap-back on error, keyboard pick-up/move/drop, a11y assertions, ~200-card list renders within performance budget)
