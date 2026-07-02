# EPIC-7: Per-board activity feed

**Goal:** Any board member (Owner, Editor, or Viewer) can see a chronological history of what changed on the board — who did it, what they did, and when — giving teams full visibility into board activity. Non-members cannot read the feed (existence not disclosed). The log is append-only and immutable: entries are written by the system as a side effect of other mutations; users cannot create, edit, or delete them.
**FRs covered:** Activity feed (§1.3, §1.5); per-board roles enforced server-side (§1.4); pagination and authorization NFRs (§1.6).
**Traces:** §1.3, §1.4, §1.5, §1.6, §2.2

## Stories
- STORY-7.1 — Record activity events (side-effect of board/list/card/label/assignee/comment mutations)
- STORY-7.2 — View a board's activity feed (paginated, reverse-chronological; any member; non-member → 404)
- STORY-7.3 — Live activity updates (new entries appear in the open feed within ~1s via EPIC-6 transport)

## Out of scope
- User-facing create/edit/delete of activity entries — the feed is strictly append-only and system-written.
- Filtering or searching within the activity feed (deferred to EPIC-8).
- Push notifications or email digests from activity events.
- Engineering observability, structured logging, Sentry error tracking — those belong to EPIC-10, not here.
- Purging or archiving old activity entries (no retention policy in v1.0).

## Notes
- Activity entries reference the actor (user id + display name at write time) and the target (entity id + a snapshot label). If the target is later deleted, the historical entry is unaffected and still renders a sensible human-readable description.
- EPIC-5 (Comments) mutations, EPIC-3 (Cards), EPIC-2 (Lists), EPIC-4 (Labels/metadata), EPIC-1 (membership changes), and board-level mutations are the event sources. EPIC-7 records them; it does not own those mutations.
- EPIC-6 owns the SignalR transport. STORY-7.3 consumes it.
