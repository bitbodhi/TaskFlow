# STORY-3.4: Move a card to another list on the same board

**Epic:** EPIC-3

As an Editor or Owner of a board, I want to move a card from one list to another list on the same board, so that I can reflect a change in the task's status or workflow stage.

## Acceptance Criteria

**Given** I am an authenticated Editor or Owner of the board **When** I PATCH `/cards/{cardId}/move` with a valid `targetListId` and an optional `afterCardId` (position hint) **Then** the card's `listId` is updated to `targetListId`, its `rank` is set to place it immediately after `afterCardId` (or appended at the end if `afterCardId` is omitted or null), I receive `200 OK` with the updated card DTO (`id`, `title`, `listId`, `rank`, `updatedAt`).

**Given** I supply a `targetListId` that belongs to a **different board** than the card, or a `targetListId` that does not exist **When** I PATCH `/cards/{cardId}/move` **Then** I receive `404` RFC 7807 with a generic message (existence of the list is not disclosed — cross-board and not-found cases return the same response) and the card is unchanged.

**Given** I supply an `afterCardId` that does not belong to `targetListId` **When** I PATCH `/cards/{cardId}/move` **Then** I receive `400` RFC 7807 with a descriptive error and the card is unchanged.

**Given** I supply a `targetListId` that is the **same** as the card's current list with no `afterCardId` **When** I PATCH `/cards/{cardId}/move` **Then** the card is appended to the end of the list (treated as a no-op reorder, not rejected); I receive `200 OK`.

**Given** I am a Viewer on the board **When** I PATCH `/cards/{cardId}/move` **Then** I receive `403` RFC 7807 and the card is unchanged.

**Given** I am not a member of the board, or present no/invalid JWT **When** I PATCH `/cards/{cardId}/move` **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated) and the card is unchanged.

**Given** I move a card in the board UI (drag from one list column to another, or use an explicit "Move to list" control) **When** the move is confirmed **Then** the card disappears from the source column and appears in the target column at the chosen position. Loading/error states are shown; on error the card snaps back to its original position.

**Given** the move UI renders **When** viewed by a Viewer **Then** the move affordance is not present or is disabled.

**Given** the move UI is displayed **When** rendered **Then** it meets WCAG 2.2 AA: all interactive controls are keyboard-accessible, selection of target list is announced to assistive technology, visible focus indicators present.

## NFR / constraints
- **Authorization:** Editor and Owner may move; Viewer receives `403`. Enforced server-side.
- **IDOR / cross-board guard:** `targetListId` is validated to belong to the same board as the card before any write, per the non-disclosure convention established in STORY-0.4. Any `targetListId` that does not exist within this card's board — whether the list does not exist at all or belongs to another board — returns a single generic `404` RFC 7807 response. The two cases are merged so that list existence on other boards is never disclosed via distinct error messages. `400` is NOT used for this guard; `404` is the correct status per the shared convention.
- **Ordering model (shared with STORY-3.5):** on move, the card's `rank` is computed as the midpoint between `afterCardId.rank` and the next card's rank (or `afterCardId.rank + 1.0` if no next card exists, or `1.0` if the target list is empty). If the midpoint gap falls below epsilon, the list's ranks are renumbered in the same transaction. Cards in each list are always returned `ORDER BY rank ASC`.
- **Atomicity:** the `listId` update and `rank` update occur in a single database transaction.
- **Real-time:** other board members will not see the move until EPIC-6 broadcasts a `card.moved` event — noted as a constraint, not authored here.
- Response: `200 OK` with updated card DTO on success; RFC 7807 on all error paths; structured logging of the move event (fromListId, toListId, cardId).

## UX spec (skeleton)
**Option A — Drag-and-drop (primary):** the card is draggable; dropping it onto a different list column calls this endpoint. Implementation shares the drag mechanism with STORY-3.5 (reorder within list). **Option B — "Move to list" menu:** accessible fallback; a dropdown in the card detail panel lists all other lists on the board; selecting one and confirming calls this endpoint (appends to end, `afterCardId` omitted). Both options must be implemented — DnD as the primary UX, the menu as the keyboard-accessible fallback. States during move: **dragging** (card ghost in motion), **submitting** (card shown in target position with opacity/spinner), **error** (card snaps back, toast error `role="alert"`), **success** (card rendered at new position, rank order reflected). On keyboard: Tab to card → Enter opens card menu → "Move to list" → select list → Enter confirms. WCAG 2.2 AA: `aria-grabbed`, `aria-dropeffect` on drag targets (or equivalent ARIA drag pattern), menu option announced on selection.

## Out of scope
- Moving a card to a list on a different board (server returns 404 per non-disclosure convention — not a UX flow).
- Moving multiple cards at once (deferred).
- Reordering within the same list using this endpoint (use STORY-3.5 for within-list reorder; this endpoint also handles same-list repositioning but it is not the primary flow for that).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.5, §1.6, §2.2
**Tests:** `MoveCardCommandHandlerTests` (Application unit — success cross-list, success same-list append, targetListId wrong board → 404 generic, targetListId not found → 404 generic (same response body as wrong-board case), afterCardId not in targetList → 400, Viewer → authz exception, non-member → not-found, rank midpoint computed correctly, gap-below-epsilon triggers renumber); `MoveCardEndpointTests` (Api integration — 200 DTO with updated listId and rank, 404 cross-board targetListId, 404 non-existent targetListId (verify both return identical RFC 7807 response shape), 400 invalid afterCardId, 401 no token, 403 Viewer, 404 non-member, atomicity verified on read-back); `CardMoveList.spec.ts` (Vitest — move-to-list dropdown renders all lists, submit calls correct endpoint, card snaps back on error, Viewer sees no move control, a11y)
