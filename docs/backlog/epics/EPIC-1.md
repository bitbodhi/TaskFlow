# EPIC-1: Board Membership & Per-Board Roles

**Goal:** An Owner can build a team on a board and precisely control what each member can do; per-board roles (Owner / Editor / Viewer) are enforced server-side on every mutation, turning EPIC-0's single-Owner board into a fully collaborative workspace.

**FRs covered:** Per-board roles & membership management (§1.4); Authentication — current-user & sign-out (§1.3); Server-enforced authorization across all mutations (§1.4, §1.6).

**Traces:** §1.3, §1.4, §1.6, §2.2

## Stories

- STORY-1.1 — View a board's members and their roles
- STORY-1.2 — Add a member to a board by email (Owner only)
- STORY-1.3 — Change a member's role (Owner only)
- STORY-1.4 — Remove a member from a board (Owner only)
- STORY-1.5 — Enforce the per-board role matrix server-side
- STORY-1.6 — Delete a board (Owner only)
- STORY-1.7 — Current-user endpoint and sign-out

## Out of scope

- Email-based invitation flow with accept/decline tokens (not a registered-user lookup).
- Board visibility settings (public/private) — all boards are private for v1.
- Transferring board ownership as a dedicated operation — handled by STORY-1.3 (promote to Owner, then demote self).
- Real-time propagation of membership changes (EPIC-6 owns SignalR stories).

## Notes

- STORY-1.5 is the authorization spine: it defines the role×action matrix and the `403` response that every subsequent epic (EPIC-2 through EPIC-9) depends on. Stories in those epics cite this convention rather than re-defining it.
- Last-Owner protection is a shared invariant enforced in the domain; STORY-1.3 and STORY-1.4 both AC it.
- `GET /me` (STORY-1.7) is a lightweight lookup by token claim — no DB write; sign-out is client-side token teardown.
