# STORY-4.2: Add / remove a label on a card

**Epic:** EPIC-4

As a board Editor or Owner, I want to add or remove a board label from a card, so that cards can be categorised using the board's shared label set.

## Acceptance Criteria

**Given** I am an Editor or Owner and a label exists on the board **When** I POST `/boards/{boardId}/cards/{cardId}/labels` with `{ "labelId": "<id>" }` **Then** the label is associated with the card, I receive `201 Created` with `{ cardId, labelId }`, and the updated card label list is visible; real-time propagation <1s (via EPIC-6).

**Given** a label is already applied to a card **When** I POST the same `labelId` again **Then** I receive RFC 7807 `400` ("Label already applied to this card") and no duplicate association is created.

**Given** I am an Editor or Owner and a label is applied to a card **When** I DELETE `/boards/{boardId}/cards/{cardId}/labels/{labelId}` **Then** the association is removed, I receive `204 No Content`, and the label no longer appears on the card; real-time propagation <1s (via EPIC-6).

**Given** the `labelId` is valid and belongs to the board but is **not currently applied** to the card **When** I DELETE `/boards/{boardId}/cards/{cardId}/labels/{labelId}` **Then** I receive RFC 7807 `404` ("Label is not applied to this card") and no change is made.

**Given** I submit a `labelId` that does not belong to the same board as the card **When** I attempt to add the label **Then** I receive RFC 7807 `400` ("Label does not belong to this board") and no association is created (prevents cross-board label injection).

**Given** I am a Viewer of the board **When** I attempt to add or remove a label from a card **Then** I receive `403 Forbidden` and no change is made.

**Given** I am not a member of the board or present no/invalid token **When** I attempt any card-label mutation **Then** I receive `404` (non-member) or `401` (unauthenticated); no change is made.

**Given** the card detail panel is open **When** it is loading, has no labels, or encounters an error **Then** appropriate loading, empty ("No labels"), and error states are shown; all controls meet WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** add and remove label require Editor or Owner role, enforced server-side. Viewer → `403`. Non-member / unauthenticated → `404` / `401`.
- **Cross-board guard:** the `labelId` must belong to the same board as the card; validated server-side (prevents IDOR).
- **Idempotency:** adding an already-applied label is `400`, not silent success (unambiguous client state).
- **API:** DTOs only at the boundary; RFC 7807 on every error path. No label data is modified — labels are managed in STORY-4.1.
- **Clean Architecture:** `CardLabel` association managed in Application use-cases; no EF/Microsoft.* in Domain.
- **Read:** `GET /boards/{boardId}/cards/{cardId}/labels` returns the list of labels currently applied to the card (non-paginated — a card's applied labels are bounded in practice; spec allows at most the board's total label count which is itself paginated and bounded).
- **Real-time:** add and remove events propagate to all board members <1s (via EPIC-6); this story defines the data contract only.

## UX spec (skeleton)
Card detail panel → **Labels** section. Displays applied labels as chips: colored swatch + name (swatch is `aria-hidden`; name is the accessible text). **Add label:** clicking "Add label" (or a `+` icon) opens a dropdown/popover listing all board labels not yet applied to the card; selecting one applies it immediately (optimistic update). **Remove:** each applied label chip has an `×` button (`aria-label="Remove label <name>"`). States: loading (skeleton chips); empty ("No labels — add one"); error banner (`role="alert"`). Keyboard: popover is focusable and keyboard-navigable; `Esc` closes it; focus returns to the trigger. Color is never the sole identifier — label name is always visible in the chip and the picker list.

## Out of scope
- Creating or editing labels (STORY-4.1).
- Filtering cards by label (EPIC-8).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.5, §1.6, §2.2
**Tests:** `AddCardLabelHandlerTests`, `RemoveCardLabelHandlerTests` (Application unit — success, duplicate add → 400, cross-board label → 400, remove label not currently applied → 404, authz denied for Viewer); `CardLabelEndpointTests` (Api integration — 201/204/400 duplicate/400 cross-board/401/403/404, DELETE not-applied label → 404); `CardLabelsSection.spec.ts` (Vue/Vitest — apply/remove label, loading/empty/error states, a11y: name always visible, `×` button aria-label)
