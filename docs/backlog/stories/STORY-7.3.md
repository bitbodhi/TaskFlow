# STORY-7.3: Live activity updates

**Epic:** EPIC-7

As a board member with the Activity feed open, I want new activity entries to appear automatically within about one second of them being recorded, so that I can see the board's live history without refreshing.

## Acceptance Criteria

**Given** I have the board's Activity feed open in my browser **When** another member (or I myself in another tab) performs any mutation that produces an activity entry **Then** the new entry appears at the top of my visible feed within ~1s, without any manual page action, using the real-time channel provided by EPIC-6.

**Given** multiple activity entries are produced in rapid succession (e.g. bulk card moves) **When** they arrive via the real-time channel **Then** each is prepended to the top of the feed in order of `occurredAt` descending; no duplicates appear; existing entries are not re-fetched from the server.

**Given** `IActivityPublisher.Publish` throws (e.g. Hub invocation fails) AFTER the mutation has committed and the activity entry has been written **When** the request completes **Then** the API still returns the original success response (e.g. `200 OK` or `201 Created`); the exception is caught and logged as a structured error (including boardId and activityEntryId); no rollback of the mutation or the activity entry occurs.

**Given** SignalR reconnects and a page-1 REST re-fetch is in-flight **When** a live `ActivityEntryPushed` event arrives for an entry that is also included in the re-fetch response **Then** the Pinia feed store shows that entry exactly once (deduplication by `id` before prepending/merging).

**Given** I am a Viewer with the Activity feed open **When** another member makes a change **Then** the new entry is pushed to my feed exactly as for Owner or Editor (role does not restrict live updates — authorization is already enforced at the Hub connection/group level by EPIC-6).

**Given** my SignalR connection is temporarily dropped and then reconnects **When** the reconnection succeeds **Then** the UI re-fetches page 1 of the feed (a full refresh of the visible window) so any missed entries during the outage are recovered; new live updates then continue from the reconnected state.

**Given** I navigate away from the Activity tab or close the board **When** my component unmounts **Then** the SignalR group subscription is released (no memory leak, no phantom updates to unmounted components).

**Given** I have NOT opened the Activity feed **When** an activity entry is produced on the board **Then** the real-time push still delivers the entry to any peer who does have the feed open; there is no side effect on other parts of the UI that have not opted in.

## NFR / constraints
- **Transport:** consumes the `board-{boardId}` SignalR group provided by EPIC-6. This story MUST NOT implement a new Hub, connection manager, or group subscription mechanism — those belong to EPIC-6. STORY-7.3 only adds: (a) the server-side `Hub` method invocation to push the new `ActivityEntryDto` on the existing group, and (b) the client-side handler that prepends the DTO to the Pinia feed store.
- **Latency target:** new entry visible to all connected board members within ~1s of the originating mutation committing (§1.2, §1.6).
- **Deduplication:** the client must guard against receiving the same `ActivityEntryDto` twice (e.g. on reconnect + re-fetch race). Deduplicate by `id` in the Pinia store before prepending.
- **No unbounded growth:** the Pinia store for live-prepended entries MUST NOT grow without bound. Cap the in-memory list at a configurable maximum (e.g. 200 entries) consistent with the pagination page-size contract; older entries remain available via the paginated REST endpoint.
- **Reconnection recovery:** on SignalR reconnect, the UI calls the REST endpoint (page 1) to recover missed entries, then resumes live updates. The REST and live paths must produce the same `ActivityEntryDto` shape.
- **Authorization:** EPIC-6 enforces group membership at the Hub level; STORY-7.3 relies on that. No additional per-push authorization check is required here.
- **Orchestration (Application layer only):** the Application command handler calls `IActivityWriter` WITHIN the unit-of-work (same `SaveChanges` as the mutation), then — AFTER commit — calls `IActivityPublisher.Publish` fire-and-forget. Both calls live in the Application layer; there is no extra orchestration layer between them. The Infrastructure `IActivityPublisher` implementation holds the Hub reference; the Application layer has no knowledge of SignalR types.
- **Clean Architecture (§2.2):** the server push is triggered from the Application layer (after the mutation commits and the activity entry is written) via an `IActivityPublisher` interface; the SignalR Hub implementation lives in Infrastructure/Api. No SignalR types in Domain or Application.
- **Publisher failure isolation:** if `IActivityPublisher.Publish` throws after commit, the exception MUST be caught and logged (structured log including boardId and activityEntryId) but MUST NOT propagate to the caller. The mutation response and the activity write are already durable; only the real-time push is lost.
- **Accessibility:** new entries prepended live are announced to screen readers via an `aria-live="polite"` region on the feed list. The announcement must not be intrusive — polite (not assertive) so it does not interrupt ongoing interactions.
- **Performance:** Hub invocations are fire-and-forget from the API's perspective; a SignalR delivery failure MUST NOT cause the mutation transaction to roll back or the API request to fail.
- **Testing:** the real-time path is integration-tested against the actual Hub (using `Microsoft.AspNetCore.SignalR.Client` in test); unit tests mock `IActivityPublisher`.

## UX spec (skeleton)

**Feed behavior on live update:**
- New entry slides or fades in at the top of the `<ol>` (CSS `transition`; respects `prefers-reduced-motion` — instant insert when reduced motion is set).
- The `aria-live="polite"` region (wrapping the feed or a separate off-screen announcer) announces: "{actorDisplayName} {description}" so screen reader users are informed without disruption.
- No toast or notification — the feed itself is the notification surface.

**Reconnect recovery UI:**
- If SignalR drops, a subtle inline banner ("Reconnecting…") appears below the feed header; it disappears automatically on reconnect, after which the feed silently re-fetches page 1.
- No user action required for reconnect or recovery.

**Reduced-motion:**
- `@media (prefers-reduced-motion: reduce)`: new entries appear instantly (no slide/fade animation).

No external Figma comp — this skeleton is the design reference.

## Out of scope
- Implementing the SignalR Hub, connection lifecycle, or group management (EPIC-6 owns this).
- Push notifications or browser notifications for activity (future).
- Live updates on parts of the UI other than the Activity feed (board column updates belong to EPIC-6's own consumers).
- Offline support or full catch-up replay beyond page-1 re-fetch on reconnect.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.2, §1.3, §1.5, §1.6, §2.2
**Tests:** `ActivityPublisherTests` (Application unit — after successful activity write, `IActivityPublisher.Publish` is called with the correct DTO; a rolled-back transaction does not call the publisher; `IActivityPublisher.Publish` throwing after commit does not propagate — the handler returns success and the exception is logged); `ActivityHubIntegrationTests` (Api integration — connected SignalR test client receives `ActivityEntryPushed` within **2 s CI wall-clock timeout** of a mutation commit; non-member client does NOT receive the push; Hub failure does not fail the HTTP mutation response); `ActivityFeedLive.spec.ts` (Vue/Vitest — simulated SignalR message prepends entry to top of feed; duplicate id received while a page-1 re-fetch is in-flight appears only once in the feed (dedup by id); `aria-live` region is present; reduced-motion class applied when `prefers-reduced-motion: reduce`; reconnect triggers REST re-fetch then resumes live updates; unmount releases subscription).
