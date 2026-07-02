# STORY-2.3: Delete a list (and its cards)

**Epic:** EPIC-2

As an Editor or Owner of a board, I want to delete a list after confirming the action, so that I can remove columns that are no longer needed, understanding that all cards inside them will also be permanently deleted.

## Acceptance Criteria

**Given** I am an Owner or Editor of a board **When** I confirm and DELETE a list **Then** the list and **all of its cards** are permanently deleted in a single transaction, and I receive `204 No Content`.

**Given** I delete a list that contains one or more cards **When** the deletion succeeds **Then** no orphaned cards remain in the database — the cascade is verified by a subsequent `GET` returning empty for that list's former `listId`.

**Given** I am an unauthenticated caller **When** I attempt to delete a list **Then** I receive `401` and the list is unchanged.

**Given** I am authenticated but not a member of the board **When** I attempt to delete a list **Then** I receive `404` (existence not disclosed to non-members) and the list is unchanged.

**Given** I am a Viewer of the board **When** I attempt to delete a list **Then** I receive `403` (member whose role forbids the mutation) and the list is unchanged.

**Given** the list id does not exist or belongs to a different board **When** I attempt to delete it **Then** I receive `404` and no change occurs.

**Given** the UI **When** I initiate deletion of a list **Then** a confirmation dialog warns me that all cards in the list will also be permanently deleted, displaying the card count N from the list DTO's `cardCount` field (no separate count endpoint is called), and deletion only proceeds on explicit confirmation.

**Given** a list is successfully deleted **When** another board member views the board **Then** real-time propagation < 1 s (via EPIC-6) removes the list and its cards from connected clients' views.

**Given** the board has only the default seeded list from EPIC-0 **When** I delete it **Then** the deletion succeeds (there is no minimum-list invariant in v1) and the board becomes empty.

## NFR / constraints

- **Security / authorization:** deletion is permitted for Owner and Editor only. Status-code convention (per EPIC-1 / STORY-1.5): `401` unauthenticated; `404` non-member or list not found; `403` Viewer; `400` validation.
- **Cascade integrity:** list + all child cards deleted in **one database transaction**; if the transaction fails, neither the list nor any cards are deleted. EF Core cascade delete (or explicit bulk delete in Application) — no orphaned rows.
- **Default list:** the list seeded by STORY-0.3 carries no special flag — it is deletable like any other list.
- **Destructive action UX:** confirmed before execution; the confirmation message explicitly names card deletion. No "undo" in v1.
- **API:** `DELETE /boards/{boardId}/lists/{listId}`; no request body; `204` on success; RFC 7807 on every error path.
- **Card count source:** the board-view list DTO (populated in STORY-2.5 / board fetch) carries a `cardCount: int` field per list. The confirmation dialog uses this client-side value — no separate count endpoint is called. The DELETE endpoint itself returns `204 No Content` with no body.
- **Clean Architecture (§2.2):** delete-list use-case in Application (deletes list + cards in one transaction; no count query needed at delete time); EF Core in Infrastructure; thin controller in Api.
- **Real-time:** propagation constraint noted; EPIC-6 owns the SignalR story.

## UX spec (skeleton)

List column header includes a **Delete list** menu item (accessible via a `⋮` options button). Clicking opens a **confirmation modal** with the message: "Delete '[List Name]'? This will permanently delete the list and all [N] cards inside it. This cannot be undone." N is sourced from the list DTO's `cardCount` field already held in client state — no additional API call is made to obtain it. Two buttons: **Delete** (destructive, red) and **Cancel**. Keyboard: focus lands on Cancel by default; `Esc` closes the dialog without deleting. On confirm, the modal enters a submitting state (both buttons disabled, spinner). On `204` the column disappears and a success toast is shown. On error, a `role="alert"` banner appears inside the modal; the modal stays open. WCAG 2.2 AA: dialog uses `role="dialog"` with `aria-labelledby` and `aria-describedby`; focus is trapped inside while open; focus returns to the trigger element on close. No external comp — this note is the design reference.

## Out of scope

- Archiving lists (hide without deleting cards).
- Soft-delete / undo / recycle bin.
- Bulk deletion of multiple lists.
- Real-time delivery story (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.5, §1.6, §2.2
**Tests:** `DeleteListHandlerTests` (Application unit — success with cascade cards, success empty list, success deletes EPIC-0 default list, authz denied Viewer, authz denied non-member, list not found, transaction rollback on DB error); `DeleteListEndpointTests` (Api integration — 204, 401, 403 Viewer, 404 non-member, 404 wrong-list-id, verify no orphaned cards after deletion); `DeleteListConfirmDialog.spec.ts` (Vue/Vitest — dialog opens on menu click, Cancel aborts, Confirm calls DELETE, submitting state, error banner, focus trap + a11y, Esc closes)
