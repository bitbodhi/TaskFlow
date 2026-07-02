# EPIC-5: Comments on cards

**Goal:** Editors and Owners can discuss a card via threaded comments; Viewers can read them. Comments are paginated, editable by their author (showing an edited indicator), and deletable by their author or by the board Owner. All mutations are enforced server-side with the standard role and status-code conventions.
**FRs covered:** Comments (§1.3, §1.5); Per-board roles — Viewer read-only, Editor+ write (§1.4); Pagination (§1.6); RFC 7807, WCAG 2.2 AA, real-time propagation (§1.6, §2.2).
**Traces:** §1.5, §1.4, §1.6, §2.2

## Stories

- STORY-5.1 — Add a comment to a card
- STORY-5.2 — View a card's comments (paginated)
- STORY-5.3 — Edit own comment
- STORY-5.4 — Delete a comment

## Out of scope

- Threaded replies / nested comments (flat list only in v1).
- Reactions / emoji on comments.
- @mentions or notifications.
- Rich-text / markdown rendering (plain text body only in v1).
- Attachments on comments.
- Real-time propagation (SignalR stories authored in EPIC-6; this epic notes the expected latency only).

## Notes

- Body length: **1–5000 characters**, validated in Domain entity, EF column constraint, and client validation — consistent across all stories.
- Status-code convention (shared, do not reinvent): `401` unauthenticated; `404` non-member (board or card existence not disclosed); `403` role forbids the mutation (Viewer attempting to write; non-author non-Owner attempting to edit/delete); `400` validation failure.
- Ownership rule: a user may edit or delete **only their own** comment; a board **Owner** may delete **any** comment on any card in that board; no other role may delete another user's comment.
- Real-time: after a successful mutating operation, other connected clients observe the change via EPIC-6 with < 1 s propagation. Stories note this expectation but do not author SignalR code.
