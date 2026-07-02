# STORY-6.4: Propagate comment mutations to board group in real time

**Epic:** EPIC-6

As a board member viewing a card's comment thread, I want new, edited, and deleted comments to appear in my view within ~1 s with no refresh, so that I can follow the discussion without polling or reloading.

## Acceptance Criteria

**Given** Member A and Member B are both viewing board X, and Member B has the comment thread of card C open **When** Member A posts a new comment on card C **Then** within ~1 s the new comment appears at the bottom of Member B's comment thread without a page reload. The two-client integration test asserts that the receiver observes the event within a **2 s CI wall-clock timeout** (validating the <1 s NFR with headroom for CI overhead).

**Given** the originating member (Member A) is in the board group **When** their own `CommentAddedEvent`, `CommentEditedEvent`, or `CommentDeletedEvent` is delivered back to their connection **Then** the SPA comment store update is idempotent — no duplicate comment appears and no existing comment is doubly removed as a result of the self-echo.

**Given** Member A edits their own comment **When** the edit is saved **Then** within ~1 s Member B sees the updated comment body and the "(edited)" indicator on that comment.

**Given** Member A (or the board Owner) deletes a comment **When** the deletion is committed **Then** within ~1 s the comment disappears from Member B's thread; if Member B had that comment focused, focus moves to the next logical element (keyboard accessibility).

**Given** Member B has the card's comment thread closed (or the card detail is not open) **When** a comment event arrives **Then** the Pinia comment store is updated silently; when Member B opens the thread they see the current state without a separate fetch.

**Given** the event arrives while Member B is offline or reconnecting **When** Member B reconnects (per STORY-6.1) **Then** the full-board reload on reconnect includes an up-to-date comment count; Member B fetches the comment thread on demand and sees current comments (no special comment replay needed).

**Given** a comment event arrives for a card that belongs to a different board than `board-{X}` **When** the SPA receives it **Then** it is silently ignored (no stale state, no error).

## NFR / constraints

- **< ~1 s propagation (§1.6):** broadcast issued via `IBoardEventPublisher` within the same request after the DB write commits; no async queue.
- **Azure SignalR Default mode / scale-out safety (§2.4, §2.9):** broadcast to group `board-{boardId}` via the Azure SignalR Service; any API instance may publish; no per-instance socket state.
- **Clean Architecture (§2.2):** `IBoardEventPublisher` interface in Application; SignalR implementation in Infrastructure via `IHubContext<BoardHub>`; EPIC-5 comment command handlers call the publisher after committing the mutation.
- **Hub authorization (EPIC-6 spine):** group `board-{boardId}` (STORY-6.1) ensures only board members receive comment events. Server-side.
- **Message shape:** typed event DTOs — `CommentAddedEvent` (commentId, cardId, boardId, authorId, authorDisplayName, body, createdAt), `CommentEditedEvent` (commentId, cardId, boardId, body, editedAt), `CommentDeletedEvent` (commentId, cardId, boardId). DTOs serialized as camelCase JSON. No raw DB entities over the wire.
- **Idempotency:** Pinia comment store applies events immutably (add/replace/remove by commentId); duplicate events (e.g. self-echo) are idempotent; events for unknown commentIds on delete are silently dropped.
- **No self-echo suppression required:** if the originating client receives its own event, the store update is idempotent.
- **Pagination consistency:** comments are paginated (EPIC-5 STORY-5.2); incoming `CommentAddedEvent` appends the comment to the currently loaded page if the thread is visible; it does not reload the full paginated list.
- **Security:** publish call is inside the already-authorized EPIC-5 command handler (STORY-1.5 role matrix); no extra auth on the publish path.
- **WCAG 2.2 AA:** live comment additions use `aria-live="polite"` in the comment thread container; deletions do not orphan keyboard focus; no color-only change signals.

## Out of scope

- Card mutations (STORY-6.2).
- List and board structural changes (STORY-6.3).
- Presence (STORY-6.5).
- Notifications when the user is not viewing the board.
- Typing indicators ("Member A is typing…").

**Traces:** §1.5, §1.6, §2.2, §2.4, §2.9
**Tests:** `CommentBroadcastTests` (Application unit — add/edit/delete comment command handlers call `IBoardEventPublisher` with correct event DTO; mock publisher verifies each call including correct cardId and boardId); `CommentRealTimePropagationTests` (Api integration — two hub clients in the same board group: Member A posts/edits/deletes a comment via REST, Member B receives the corresponding SignalR event within a **2 s CI wall-clock timeout**; client in a different board group does not receive; self-echo to Member A does not produce a duplicate comment in the Pinia store); `commentStore.spec.ts` (Vitest — `CommentAddedEvent` appends to store, `CommentEditedEvent` updates body and sets edited flag, `CommentDeletedEvent` removes entry, duplicate add is idempotent, delete of unknown id is no-op, applying the same edit event twice yields the same final state)
