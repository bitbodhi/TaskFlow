# STORY-1.1: View a board's members and their roles

**Epic:** EPIC-1

As a board member (any role), I want to see the full list of members on a board and their roles, so that I understand who has access and what they can do.

## Acceptance Criteria

**Given** I am an authenticated member of a board (Owner, Editor, or Viewer) **When** I request `GET /boards/{boardId}/members` (with optional `page` / `pageSize`) **Then** I receive `200 OK` with a paginated envelope `{ items, page, pageSize, total }` where each item contains the member's user id, display name, email, role, and the date they were added.

**Given** I am authenticated but not a member of the board **When** I request `GET /boards/{boardId}/members` **Then** I receive `404` RFC 7807 (board existence not disclosed to non-members).

**Given** I am unauthenticated **When** I request `GET /boards/{boardId}/members` **Then** I receive `401` RFC 7807.

**Given** the board has more than `pageSize` members **When** I request page 2 **Then** I receive the correct slice, with `total` accurately reflecting the full member count.

**Given** the members panel is loading **When** data is in flight **Then** a visible loading indicator is shown and the list area is not blank.

**Given** the board has at least one member **When** the data loads **Then** each member row shows display name, email, and role badge; the list is accessible and meets WCAG 2.2 AA (role badge has sufficient contrast, row is keyboard-navigable, name is not conveyed by colour alone).

**Given** the fetch fails (network or 5xx) **When** the panel is open **Then** an error state with a retry action is displayed and a `role="alert"` message is announced to screen-reader users.

## NFR / constraints

- **Security / authorization:** read is permitted for all three roles; `404` (not `403`) for non-members (HTTP status convention from STORY-0.3/EPIC-0: `401` unauthenticated, `404` non-member, `403` wrong-role-for-action).
- **Performance / pagination:** offset-based pagination envelope (`page`, `pageSize` default 20 max 100) consistent with the shared envelope in EPIC-0.
- **API:** DTOs only at the boundary; RFC 7807 on every error path.
- **Clean Architecture:** query handler in Application; EF Core projection in Infrastructure; thin controller in Api. No EF / Microsoft.* types in Domain.
- **Accessibility:** WCAG 2.2 AA; role badges must not rely solely on colour.
- **Real-time:** membership changes made by another Owner propagate <1s (via EPIC-6) — this story does not author SignalR code.

## UX spec (skeleton)

Members panel (sidebar or modal) on the board view. States: **loading** (spinner/skeleton rows), **populated** (scrollable list of member rows — avatar/initials, display name, email, role badge), **empty** (only possible for a brand-new board before this epic, not reachable post-1.2), **error** (message + retry button, `role="alert"`). Role badge colours must meet WCAG 2.2 AA contrast; provide a text label alongside any colour indicator. Panel is keyboard-navigable (Tab through rows); no destructive action in this story.

## Out of scope

- Adding, removing, or changing roles (STORY-1.2, 1.3, 1.4).
- Pagination UI controls (infinite scroll or page buttons) — the API contract is defined here; UI pagination controls may be deferred.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.6, §2.2
**Tests:** `GetBoardMembersHandlerTests` (Application unit — success paginated, non-member → throws NotFound, unauthenticated → throws Unauthenticated); `GetBoardMembersEndpointTests` (Api integration — 200 paginated envelope, 401, 404 non-member, pagination slice accuracy); `BoardMembersPanel.spec.ts` (Vue/Vitest — loading state, populated list, error state + retry, role badge text, a11y)
