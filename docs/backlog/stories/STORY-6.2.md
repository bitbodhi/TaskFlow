# STORY-6.2: Propagate card mutations to board group in real time

**Epic:** EPIC-6

As a board member viewing a board, I want card changes made by other members (create, edit, move between lists, reorder, delete, and metadata updates) to appear in my view within ~1 s with no refresh, so that I always see the current state of the board without manual intervention.

## Acceptance Criteria

**Given** Member A and Member B are both viewing board X (both in group `board-{X}`) **When** Member A creates a card **Then** within ~1 s Member B's board view adds the card to the correct list in the correct position without a page reload; Member A's own view is not double-updated (the mutation already applied locally). The two-client integration test asserts that the receiver observes the event within a **2 s CI wall-clock timeout** (validating the <1 s NFR with headroom for CI overhead).

**Given** Member A is in the board group and performs a mutation **When** their own broadcast event is delivered back to their connection **Then** the SPA store update is idempotent — no duplicate card, list, or comment appears as a result of the self-echo.

**Given** Member A edits a card's title **When** the change is saved **Then** within ~1 s Member B sees the updated title on the card.

**Given** Member A moves a card from List A to List B **When** the move is persisted **Then** within ~1 s Member B sees the card removed from List A and appear in List B at the correct position.

**Given** Member A reorders cards within a list **When** the new positions are persisted **Then** within ~1 s Member B's list reflects the new order.

**Given** Member A deletes a card **When** the deletion is committed **Then** within ~1 s Member B's view removes that card from the list with no error state.

**Given** Member A updates a card's metadata (label, assignee, due date, or priority — EPIC-4) **When** the change is persisted **Then** within ~1 s Member B sees the updated metadata on the card.

**Given** the propagated event arrives while Member B is offline or reconnecting **When** Member B reconnects (per STORY-6.1) **Then** the full-board reload on reconnect retrieves all changes missed during the outage; no real-time messages are lost or cause stale state.

**Given** a card mutation event arrives for a board the client has left **When** the SPA receives it **Then** it is silently ignored (no stale state, no error).

## NFR / constraints

- **< ~1 s propagation (§1.6):** from the moment the API persists the mutation to the moment the SignalR message is delivered to connected clients. The broadcast is issued by the Application layer (via `IBoardEventPublisher`) within the same request after the DB write commits; no async queue introduces extra latency.
- **Azure SignalR Default mode / scale-out safety (§2.4, §2.9):** broadcast is addressed to group `board-{boardId}` via the Azure SignalR Service; any API instance can issue it; no per-instance socket state.
- **Clean Architecture (§2.2):** the `IBoardEventPublisher` interface lives in Application; its SignalR implementation lives in Infrastructure (`IHubContext<BoardHub>`); command handlers call the publisher — no hub-to-handler dependency inversion.
- **Hub authorization (§EPIC-6 spine):** only members of the board receive messages in `board-{boardId}`; server-side group membership (STORY-6.1) enforces this. No client-side filtering needed.
- **Message shape:** each event carries a typed, versioned DTO (e.g. `CardCreatedEvent`, `CardMovedEvent`, `CardDeletedEvent`, `CardMetadataUpdatedEvent`). DTOs are defined in Application and serialized as camelCase JSON. No raw DB entities over the wire.
- **Idempotency / last-write-wins:** no OT or CRDT; the last persisted write wins. The SPA applies each incoming event to the Pinia store immutably (replace-in-place by id); stale events for deleted cards are silently dropped.
- **No self-echo:** the broadcaster does not need to suppress delivery to the originating connection (Azure SignalR handles broadcasting to the group; if the originating client is in the group it will receive the event — the SPA store update must be idempotent to handle this).
- **Security / authorization:** the publish call is inside the command handler, which already enforces role-based authorization (STORY-1.5); no additional auth on the publish path.
- **WCAG 2.2 AA:** live card additions/removals/updates do not steal keyboard focus; an `aria-live="polite"` region announces significant changes (e.g. "Card 'Deploy infra' moved to Done").

## NFR / constraints (continued)

- Card metadata fields propagated: label assignments, assignee, due date, priority (all EPIC-4 mutations). Each has its own event type to allow targeted store patching.
- Events covered in this story: `CardCreated`, `CardTitleUpdated`, `CardMoved`, `CardReordered`, `CardDeleted`, `CardLabelAssigned`, `CardLabelRemoved`, `CardAssigneeChanged`, `CardDueDateChanged`, `CardPriorityChanged`.

## Out of scope

- List structural changes (STORY-6.3).
- Comment changes (STORY-6.4).
- Presence (STORY-6.5).
- Optimistic UI updates on the originating client (those are handled by the Pinia store write in the mutation path, not by SignalR).
- Conflict UI beyond last-write-wins.

**Traces:** §1.2, §1.5, §1.6, §2.2, §2.4, §2.9
**Tests:** `CardBroadcastTests` (Application unit — each command handler calls `IBoardEventPublisher` with the correct event type and payload; mock publisher verifies call); `CardRealTimePropagationTests` (Api integration — two hub clients in the same board group: Member A triggers create/edit/move/delete/metadata via REST, Member B receives the corresponding SignalR event within a **2 s CI wall-clock timeout**; wrong-board group member does not receive the event; self-echo to originating client does not produce duplicate entries in the Pinia store); `cardStore.spec.ts` (Vitest — each incoming event type patches the Pinia store correctly; unknown card-id delete is a no-op; stale metadata event is idempotent; applying the same event twice is idempotent)
