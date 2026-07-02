# EPIC-6: Real-time Collaboration (Azure SignalR)

**Goal:** When any member mutates a board (creates/edits/moves/deletes a list, card, metadata field, or comment; renames the board; changes membership), every other member currently viewing that board sees the change within ~1 s — delivered over Azure SignalR Service (Default mode) in a way that keeps the API stateless and horizontally scalable.
**FRs covered:** Real-time collaboration (§1.2, §1.5); Azure SignalR Default-mode architecture (§2.4, §2.9); Stateless API scale-out (§2.1, §2.3 D5); < ~1 s propagation NFR (§1.6); WCAG 2.2 AA (§1.6).
**Traces:** §1.2, §1.5, §1.6, §2.2, §2.4, §2.9

## Stories

- STORY-6.1 — Establish an authenticated real-time connection (negotiate + hub authorization + SPA lifecycle)
- STORY-6.2 — Propagate card mutations to board group in real time (create/edit/move/reorder/delete + metadata)
- STORY-6.3 — Propagate list and board structural changes to board group in real time
- STORY-6.4 — Propagate comment mutations to board group in real time
- STORY-6.5 — Presence: show which members are currently viewing the board

## Out of scope

- Push notifications, email notifications, or mobile alerts for changes when the user is not viewing the board.
- Durable event delivery / guaranteed-delivery queues (fire-and-forget broadcast over SignalR only in v1).
- Real-time propagation of changes outside the board being viewed (global inbox / cross-board updates — EPIC-9 My Tasks and EPIC-7 activity-feed rendering ride on EPIC-6 infrastructure but author their own subscription logic).
- Per-field collaborative locking or operational-transform conflict resolution (last-write-wins; the UI reflects the latest broadcast state).
- Presence outside the current board view (global online indicator, org-wide "who's online").

## Notes

- **Azure SignalR Default mode (§2.4, §2.9):** clients negotiate via `POST /hubs/board/negotiate`, then connect **directly** to the Azure SignalR Service. The API never holds client WebSocket connections. All API instances are equivalent (scale-out-safe); no sticky sessions.
- **Hub authorization spine:** the hub rejects unauthenticated connections (`401`) at `OnConnectedAsync`. Group-join (`JoinBoardAsync`) rejects callers who are not a MEMBER of the requested board (analogous to 404/403 — analogous semantics from STORY-0.3/STORY-1.5). Server-side; client cannot bypass it.
- **Broadcast pattern:** each mutating command handler (Application layer) publishes a domain event; an `IBoardEventPublisher` infrastructure service translates it to a SignalR group broadcast on the per-board group `board-{boardId}`. The API handler never calls the hub directly (clean-architecture dependency rule: deps inward only).
- **Mutations covered:** EPIC-2 (list CRUD/reorder), EPIC-3 (card CRUD/move/reorder), EPIC-4 (card metadata), EPIC-5 (comments). EPIC-1 membership changes (board rename, member add/change/remove) are wired in STORY-6.3.
- **Status-code convention (inherited from STORY-0.3/STORY-1.5):** `401` unauthenticated; hub group-join rejected if not a board member (no connection dropped, but `JoinBoard` returns an error); `403` not used directly on the hub (board membership is binary — member vs. non-member).
- STORY-6.5 (Presence) is optional and explicitly scoped so it can be deferred without blocking 6.1–6.4.
