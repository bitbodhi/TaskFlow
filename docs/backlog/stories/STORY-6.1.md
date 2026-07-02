# STORY-6.1: Establish an authenticated real-time connection

**Epic:** EPIC-6

As a board member, I want my browser to silently establish a real-time connection when I open a board and join that board's live-update group, so that I receive changes from other members instantly without refreshing.

## Acceptance Criteria

**Given** I am an authenticated board member **When** the board view mounts **Then** the SPA calls `POST /hubs/board/negotiate` (with my JWT), receives Azure SignalR connection info, and the `@microsoft/signalr` client connects directly to the Azure SignalR Service — the API holds no WebSocket socket.

**Given** the SignalR connection is established **When** the SPA sends `JoinBoard(boardId)` via the hub **Then** the hub verifies server-side that the caller is a MEMBER of `boardId`; if valid the caller is added to group `board-{boardId}`; if not a member the hub returns an error and the client is not added to the group.

**Given** I am unauthenticated (no token or expired token) **When** a connection is attempted **Then** the hub's `OnConnectedAsync` rejects the connection with `401`; no group is joined.

**Given** I am authenticated and in the board group **When** the SPA loses connectivity (network drop, container restart) **Then** the `@microsoft/signalr` client reconnects with exponential backoff (using `.withAutomaticReconnect()`); upon reconnect it re-sends `JoinBoard(boardId)` to re-join the group; the board view displays a transient "Reconnecting…" banner during the outage.

**Given** the reconnect banner is visible **When** the connection is restored **Then** the banner disappears and the board refreshes its data (one `GET /boards/{id}` full reload) to reconcile any changes missed during the outage.

**Given** I navigate away from the board **When** the board view unmounts **Then** the SPA sends `LeaveBoard(boardId)`, calls `connection.stop()`, and no further messages are received for that board.

**Given** the SPA is loading the board and establishing the connection **When** the connection is not yet `Connected` **Then** a non-blocking loading indicator is shown in the board header (e.g. a small spinner or "Connecting…" badge); the board's cached REST data renders immediately if available.

## NFR / constraints

- **Azure SignalR Default mode (§2.4, §2.9):** negotiate endpoint routes through the API; the client then connects directly to the service. The API holds no client sockets; all API instances are equivalent (scale-out-safe; no sticky sessions required).
- **Hub authorization (EPIC-6 spine):** `OnConnectedAsync` validates the JWT via the standard ASP.NET Core `[Authorize]` attribute; `JoinBoardAsync` performs a server-side membership check (calls `IBoardMembershipRepository` or equivalent) before adding the caller to `board-{boardId}`. These checks are not bypassable client-side.
- **Clean Architecture:** the hub lives in the Api layer and calls Application-layer services (no direct DB access in the hub; deps inward only).
- **Scale-out safety:** no per-instance in-memory group state; Azure SignalR Service manages group membership — safe across multiple API instances.
- **SPA connection lifecycle:** connect on board-view mount; reconnect with backoff; `LeaveBoard` + `stop()` on unmount. Managed in a composable (e.g. `useBoardHub`), not scattered across components.
- **Security:** JWT forwarded as a query-string token on the WebSocket handshake (standard SignalR pattern); connection info from negotiate must not be cached client-side beyond the session.
- **Performance:** negotiate round-trip must not add perceptible latency to the board-load critical path; connection is initiated in parallel with the initial board data fetch.
- **WCAG 2.2 AA:** the "Reconnecting…" banner uses `role="status"` or `aria-live="polite"`; the "Connecting…" badge is accessible; no color-only status signaling.

## UX spec (skeleton)

Board header area: small **connection-status badge** (icon + text) with three states — "Connecting…" (spinner, `aria-live="polite"`), hidden/no badge when `Connected`, "Reconnecting…" (amber, `role="status"`) on transient disconnect, "Offline" (red) if reconnect fails after max retries. On reconnect → full board data refresh (no user action required). No modal, no blocking overlay. Badge is keyboard-focusable if interactive (e.g. "Click to retry"). No external comp — this note is the design reference.

## Out of scope

- Broadcasting mutations (STORY-6.2–6.4).
- Presence (which members are viewing — STORY-6.5).
- Hub-to-client messages about board events (this story wires the connection only).
- Push notifications when not viewing the board.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.2, §1.5, §1.6, §2.4, §2.9
**Tests:** `BoardHubAuthorizationTests` (Api integration — hub rejects unauthenticated connections with 401; hub rejects `JoinBoard` for non-member; hub accepts `JoinBoard` for valid member and adds to group `board-{boardId}`); `NegotiateEndpointTests` (Api integration — 401 without token; 200 + connection-info with valid JWT); `useBoardHub.spec.ts` (Vitest — connect on mount, `JoinBoard` called, `LeaveBoard` + stop on unmount, reconnect backoff triggers re-join, reconnecting banner shown/hidden, full reload on reconnect)
