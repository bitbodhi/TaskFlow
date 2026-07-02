# EPIC-0: Walking Skeleton — register → board → card → persist

**Goal:** A new user can register, log in, create a single board, add one card to it, and find that card still there after reloading or signing back in — proving the whole stack (Vue SPA → REST API → EF Core → SQL Server, under Clean Architecture with JWT auth) end-to-end through the thinnest possible vertical slice.
**FRs covered:** Authentication & accounts (§1.3, §1.4 self-as-Owner only); Boards — minimal create + read; Cards — minimal create + read. No lists UI, labels, comments, real-time, activity, search, drag-and-drop, or member management.
**Traces:** §1.1, §1.2, §1.3, §1.6, §2.2

## Stories
- STORY-0.1 — Register an account
- STORY-0.2 — Log in and receive a session
- STORY-0.3 — Create a board
- STORY-0.4 — Add a card to a board (persists)

## Out of scope (deferred to later epics)
- Per-board roles/membership beyond "creator is Owner" (§1.4), real-time sync (§1.5), lists as a managed feature, labels/assignees/due dates/priorities, comments, activity feed, "My Tasks", search/filter, drag-and-drop ordering.
- Email verification, password reset, social/OAuth login, account lockout/rate-limiting.

## Notes for downstream stories
- To give a card a home without introducing list management, **STORY-0.3 seeds each new board with one default list**; STORY-0.4 attaches the card to it. The board→list→card hierarchy (§1.1) is honored in the data model even though lists have no UI yet.
- Every mutating endpoint (create board, create card) is an **authenticated** request with **server-side authorization** (the caller must be the board's Owner). Register/login are the pre-auth boundary.
