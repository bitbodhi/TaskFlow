# STORY-6.3: Propagate list and board structural changes to board group in real time

**Epic:** EPIC-6

As a board member viewing a board, I want list mutations (create, rename, delete, reorder) and board-level structural changes (board rename, member added/role changed/removed) to appear in my view within ~1 s with no refresh, so that the board structure I see is always current.

## Acceptance Criteria

**Given** Member A and Member B are both viewing board X **When** Member A creates a new list **Then** within ~1 s the new list column appears in Member B's board view at the correct position. The two-client integration test asserts that the receiver observes the event within a **2 s CI wall-clock timeout** (validating the <1 s NFR with headroom for CI overhead).

**Given** Member A is in the board group and performs a list or board structural mutation **When** their own broadcast event is delivered back to their connection **Then** the SPA store update is idempotent — no duplicate list, member entry, or board name update appears as a result of the self-echo.

**Given** Member A renames a list **When** the rename is persisted **Then** within ~1 s the list header updates in Member B's view.

**Given** Member A deletes a list (and its cards cascade) **When** the deletion is committed **Then** within ~1 s the list column and all its cards disappear from Member B's view.

**Given** Member A reorders lists **When** the new positions are persisted **Then** within ~1 s the list order reflects the change in Member B's view.

**Given** the board Owner renames the board **When** the rename is saved **Then** within ~1 s Member B sees the updated board name in the board header and browser tab title.

**Given** the board Owner adds a new member (EPIC-1 STORY-1.2) **When** the add is persisted **Then** within ~1 s all existing connected members see the updated member count/list (if the members panel is open). The newly added member is not yet in the SignalR group — they must open the board and call `JoinBoard` (STORY-6.1) to start receiving updates.

**Given** the board Owner changes a member's role (EPIC-1 STORY-1.3) **When** the change is persisted **Then** within ~1 s existing connected members (including the affected member) receive a `BoardMemberRoleChanged` event; the affected member's local permissions are refreshed from the store (UI re-evaluates which actions are available to them).

**Given** the board Owner removes a member (EPIC-1 STORY-1.4) **When** the removal is persisted **Then** within ~1 s the removed member receives a `BoardMemberRemoved` event targeted at their user (or broadcast to the group before the hub removes them from the group); the removed member's SPA navigates to the board list with a "You have been removed from this board" toast; other connected members see the membership update.

**Given** the propagated event arrives while Member B is offline or reconnecting **When** Member B reconnects (per STORY-6.1) **Then** the full-board reload on reconnect retrieves all changes missed during the outage.

## NFR / constraints

- **< ~1 s propagation (§1.6):** broadcast issued via `IBoardEventPublisher` within the same request after the DB write commits.
- **Azure SignalR Default mode / scale-out safety (§2.4, §2.9):** group `board-{boardId}` broadcast; any API instance can publish; no per-instance socket state.
- **Clean Architecture (§2.2):** `IBoardEventPublisher` interface in Application; SignalR implementation in Infrastructure via `IHubContext<BoardHub>`; command handlers call the publisher post-commit.
- **Hub authorization (EPIC-6 spine):** group membership (STORY-6.1) ensures only board members receive events. Server-side; client cannot bypass.
- **Removed-member handling:** the Infrastructure `IBoardEventPublisher` implementation (which holds `IHubContext<BoardHub>`) calls `Groups.RemoveFromGroupAsync` for every connection belonging to the removed user **immediately after** broadcasting `BoardMemberRemovedEvent` to the group. The Application layer has no knowledge of connection ids — the eviction call lives entirely in the Infrastructure publisher. After eviction the removed user may no longer receive events for that board.
- **Message shape:** typed event DTOs (e.g. `ListCreatedEvent`, `ListRenamedEvent`, `ListDeletedEvent`, `ListsReorderedEvent`, `BoardRenamedEvent`, `BoardMemberAddedEvent`, `BoardMemberRoleChangedEvent`, `BoardMemberRemovedEvent`) serialized as camelCase JSON. No raw DB entities over the wire.
- **Idempotency / last-write-wins:** Pinia store applies events immutably (replace/add/remove by id); stale events for deleted lists or removed members are silently dropped.
- **Role-change client effect:** `BoardMemberRoleChangedEvent` carries the new role; Pinia board store updates the local user's `myRole` if the event targets the current user. The UI re-evaluates button/action visibility reactively.
- **WCAG 2.2 AA:** live list additions/removals and board-name updates do not steal focus; an `aria-live="polite"` region announces changes; the "removed from board" toast is announced via `role="alert"`.
- **Security:** publish call is inside the already-authorized command handler (STORY-1.5 role matrix); no extra auth on the publish path.
- **Group-eviction call site (Clean Architecture):** `Groups.RemoveFromGroupAsync` MUST be called from the Infrastructure `IBoardEventPublisher` implementation, not from any Application layer class. The Application layer issues no SignalR calls and holds no reference to `IHubContext` or connection ids.

## Out of scope

- Card mutations (STORY-6.2).
- Comment mutations (STORY-6.4).
- Presence (STORY-6.5).
- Pushing the removed member's browser to a login page (navigate-to-board-list is sufficient).
- Real-time propagation of board deletion (edge case: if the board is deleted, members still viewing it will see a REST error on the next interaction and navigate away).

**Traces:** §1.2, §1.5, §1.6, §2.2, §2.4, §2.9
**Tests:** `ListBroadcastTests` (Application unit — create/rename/delete/reorder list command handlers call `IBoardEventPublisher` with correct event type; mock publisher verifies call); `BoardStructureBroadcastTests` (Application unit — board-rename and EPIC-1 membership command handlers call publisher for each event type); `ListRealTimePropagationTests` (Api integration — two hub clients: Member A mutates list, Member B receives event within a **2 s CI wall-clock timeout**; wrong-board client does not receive; self-echo to Member A does not produce a duplicate list in the Pinia store); `BoardMemberRealTimeTests` (Api integration — role-change event delivered to affected member within the 2 s timeout; after `BoardMemberRemovedEvent` is broadcast, the Infrastructure publisher calls `Groups.RemoveFromGroupAsync` for the removed user's connections before the test asserts they no longer receive group messages); `boardStore.spec.ts` (Vitest — each structural event type patches the Pinia store correctly; removed-member event navigates current user to board list if self-targeted; applying the same structural event twice is idempotent)
