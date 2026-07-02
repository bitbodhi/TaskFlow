# EPIC-2: Lists CRUD & Reorder

**Goal:** Editors and Owners can structure a board into multiple ordered lists (columns) — creating, renaming, deleting, and reordering them — making lists first-class managed entities beyond the single seeded default list from EPIC-0.

**FRs covered:** Lists (§1.5); Boards → Lists → Cards hierarchy (§1.1); Server-enforced per-board roles — Owner/Editor mutate, Viewer read-only (§1.4, §1.6).

**Traces:** §1.1, §1.5, §1.6, §2.2

## Stories

- STORY-2.1 — Create a list on a board
- STORY-2.2 — Rename a list
- STORY-2.3 — Delete a list (and its cards)
- STORY-2.4 — Reorder lists on a board
- STORY-2.5 — View a board's lists in order

## Out of scope

- Moving cards between lists (EPIC-3).
- Real-time propagation of list changes to other connected clients (EPIC-6 owns SignalR stories).
- Archiving lists without deletion.
- List-level permissions (all per-board roles apply uniformly).

## Notes

- The default list seeded by STORY-0.3 behaves identically to any user-created list — same rename, delete, and reorder rules apply.
- The position/rank model (a `decimal` fractional-index per STORY-2.4) is adopted by STORY-2.1 so every new list carries a rank from creation.
- Deletion of a non-empty list cascades to its cards in the same transaction; callers are warned in the API response contract and the UI confirmation dialog.
- Real-time propagation of list mutations is a constraint noted in each story but the SignalR stories are authored in EPIC-6.
