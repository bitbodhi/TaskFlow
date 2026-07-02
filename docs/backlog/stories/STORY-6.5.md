# STORY-6.5: Presence — show which members are currently viewing the board (optional)

**Epic:** EPIC-6

As a board member, I want to see avatar indicators for other members who are currently viewing the same board, so that I know when I am collaborating in real time and can expect to see their changes appear live.

> **Optional story.** Presence is best-effort and event-driven. There is no server-side authoritative presence snapshot (Azure SignalR Default mode exposes no enumerable-group API). A late-joining member sees only members who announce themselves after it joins. A Redis-backed `IPresenceStore` behind the existing `ICacheService` seam would provide an authoritative snapshot but is explicitly out of scope (free tier, no Redis).

## Acceptance Criteria

**Given** Member A is viewing board X **When** Member B opens board X and the hub calls `JoinBoard` **Then** the hub broadcasts a `MemberJoinedPresence` event (userId, displayName, avatarUrl, boardId) to the `board-{X}` group; within ~1 s Member A's SPA receives the event and adds Member B's avatar to the presence bar.

**Given** Member B navigates away from board X and the SPA calls `LeaveBoard(boardId)` **When** the hub receives the call **Then** the hub broadcasts a `MemberLeftPresence` event (userId, boardId) to the `board-{X}` group; within ~1 s Member A's presence bar removes Member B's avatar. (Best-effort: this requires a clean `LeaveBoard` call.)

**Given** a member's connection drops ungracefully (network failure, not a clean `LeaveBoard`) **When** the SignalR keep-alive times out and `OnDisconnectedAsync` fires **Then** the hub broadcasts `MemberLeftPresence` for that user; Member A's presence bar removes the departed avatar within ~5 s of the timeout.

**Given** Member B reconnects after a drop **When** `JoinBoard` is called again (per STORY-6.1 reconnect logic) **Then** the hub broadcasts `MemberJoinedPresence` again and Member B reappears in Member A's presence bar.

**Given** multiple members are viewing the board **When** there are more than N avatars (configurable, default 5) **Then** the presence bar shows N avatars plus a "+K others" overflow indicator; all members are listed in an accessible tooltip/popover on that indicator.

**Given** I am viewing my own avatar in the presence bar **When** I inspect the presence bar **Then** my own avatar is visually distinguished (e.g. "You" label, ring) so I can distinguish myself from others; my avatar is always shown (the SPA adds itself on `JoinBoard` success without waiting for a server echo).

**Given** I am a Viewer **When** I view the presence bar **Then** I see all members who have announced themselves after I joined; presence is a read-only feature available to all roles.

**Known limitation:** a late-joiner accumulates presence only from `MemberJoinedPresence` / `MemberLeftPresence` events that arrive after it connects. Members who were already viewing the board before the late-joiner connected are not visible until they perform another action that re-broadcasts their presence (e.g. a future heartbeat, out of scope now). This is the accepted trade-off of best-effort event-driven presence without a Redis-backed snapshot.

## NFR / constraints

- **Best-effort, event-driven only:** presence state is accumulated on each client from `MemberJoinedPresence` and `MemberLeftPresence` events. There is no server-generated authoritative snapshot. The SPA initialises its own presence entry on `JoinBoard` success; it does not rely on a server-side group enumeration.
- **No `BoardPresenceSnapshot`:** the hub does NOT send a snapshot of currently connected members on join. Azure SignalR Default mode provides no group-membership enumeration API. Any implementation that calls such an API is a bug.
- **Future upgrade path (out of scope):** a Redis-backed `IPresenceStore` behind the existing `ICacheService` seam would enable the hub to emit a `BoardPresenceSnapshot` on `JoinBoard`. This is deferred until a paid tier with Redis is available.
- **Best-effort latency:** presence join broadcast is < ~1 s; graceful leave broadcast is < ~1 s; ungraceful-disconnect leave is < ~5 s (bounded by SignalR keep-alive timeout).
- **Azure SignalR Default mode / scale-out safety (§2.4, §2.9):** `MemberJoinedPresence` / `MemberLeftPresence` are broadcast to group `board-{boardId}` from the hub layer; no per-instance in-memory presence list.
- **Clean Architecture (§2.2):** presence events are broadcast directly from the hub (`BoardHub`) — presence is a connection concern, not a domain mutation. This is the one case where the hub directly sends a group message without going through the Application layer.
- **Hub authorization (EPIC-6 spine):** `JoinBoard` already validates board membership (STORY-6.1); presence broadcast piggybacks on that — no separate auth step.
- **Message shape:** `MemberJoinedPresence` DTO (userId, displayName, avatarUrl, boardId) and `MemberLeftPresence` DTO (userId, boardId). Serialized as camelCase JSON.
- **SPA presence store:** a `presenceStore` (Pinia) holds a `Map<boardId, Set<UserPresence>>`. On `JoinBoard` success the SPA adds its own entry immediately. `MemberJoinedPresence` adds; `MemberLeftPresence` removes. Duplicate `MemberJoinedPresence` for the same userId is idempotent (no double avatar). On reconnect, the SPA clears its local set and re-adds itself, then accumulates new events (no snapshot to rebuild from).
- **No persistence:** presence is ephemeral; no DB writes; no history.
- **Privacy:** only board members may see the presence bar (enforced via group membership — only board members are in `board-{boardId}`). No leaking to non-members.
- **WCAG 2.2 AA:** avatar images have `alt` text (member display name); overflow popover is keyboard-reachable (`role="tooltip"` or disclosure button); presence bar does not steal focus; `aria-live="polite"` announces join/leave to screen readers (e.g. "Alex joined the board").

## UX spec (skeleton)

Board header — **presence bar** (right side, before the Members button): a horizontal strip of avatar circles (32 px, with 1 px ring for self). Overflow: 5+ → show 5 avatars + "+K" badge. Hover/focus on any avatar: tooltip with display name. Hover/focus on "+K": popover listing all overflow members. Self-avatar: subtle ring + "You" label on hover. Join/leave transitions: fade-in / fade-out (200 ms, `prefers-reduced-motion` respected). No external comp — this note is the design reference.

## Out of scope

- Per-card presence ("Member A is editing this card").
- Typing indicators in comment threads.
- Presence history or "last seen" timestamps.
- Presence outside the current board view (global "who's online").
- Push notifications triggered by presence.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.2, §1.5, §1.6, §2.4, §2.9
**Tests:** `PresenceBroadcastTests` (Api integration — Member A joins board, Member B joins board: Member A receives `MemberJoinedPresence` for B within a **2 s CI wall-clock timeout**; Member B calls `LeaveBoard`: Member A receives `MemberLeftPresence`; non-member cannot trigger presence broadcast; no `BoardPresenceSnapshot` is ever sent); `presenceStore.spec.ts` (Vitest — `MemberJoinedPresence` adds to set, `MemberLeftPresence` removes, duplicate `MemberJoinedPresence` for same userId is idempotent, reconnect clears set and re-adds self, overflow threshold renders "+K" indicator, late-joiner does NOT receive a snapshot and starts with only self in set); `PresenceBar.spec.ts` (Vitest — renders up to N avatars, overflow badge shows correct count, self-avatar distinguished, aria-live announces join/leave, keyboard-reachable popover)
