# STORY-1.5: Enforce the per-board role matrix server-side

**Epic:** EPIC-1

As a product stakeholder, I want every board mutation to be gated by the caller's per-board role, so that Editors can collaborate freely while Viewers remain read-only and only Owners can manage membership and delete the board — with no way to bypass these rules from the client.

## Acceptance Criteria

**Role × action matrix — every row is a testable AC:**

| Action category | Owner | Editor | Viewer | Non-member | Unauth |
|---|---|---|---|---|---|
| Read board / members / lists / cards / comments | ✅ | ✅ | ✅ | 404 | 401 |
| Create / edit / move / delete **lists** | ✅ | ✅ | 403 | 404 | 401 |
| Create / edit / move / delete **cards** | ✅ | ✅ | 403 | 404 | 401 |
| Add / edit **own comment** (Editor or Owner role required; Viewer cannot comment) | ✅ | ✅ | 403 | 404 | 401 |
| Delete **own comment** (author-only for Editors) | ✅ | ✅ | 403 | 404 | 401 |
| Delete **any comment** (board Owner may delete any comment, including another Editor's) | ✅ | 403 | 403 | 404 | 401 |
| **Card metadata mutations** — set/clear due date, set/clear priority, add/remove labels on a card, assign/unassign members (EPIC-4 cites this row) | ✅ | ✅ | 403 | 404 | 401 |
| Manage **labels** (create / rename / delete) | ✅ | ✅ | 403 | 404 | 401 |
| Add **board members** (STORY-1.2) | ✅ | 403 | 403 | 404 | 401 |
| Change member **role** (STORY-1.3) | ✅ | 403 | 403 | 404 | 401 |
| Remove **member** (STORY-1.4) | ✅ | 403 | 403 | 404 | 401 |
| **Delete board** (STORY-1.6) | ✅ | 403 | 403 | 404 | 401 |

**Given** a Viewer calls any mutating endpoint (list/card/comment/label create, edit, move, or delete) **When** the request reaches the server **Then** the server returns `403` RFC 7807 before any data is written, regardless of the request body.

**Given** an Editor calls an Owner-only endpoint (add/change/remove member, delete board) **When** the request reaches the server **Then** the server returns `403` RFC 7807 before any data is written.

**Given** an authenticated non-member calls any board endpoint **When** the request reaches the server **Then** the server returns `404` RFC 7807 (existence not disclosed to non-members) before any data is read or written.

**Given** an unauthenticated caller reaches any board endpoint **When** the request reaches the server **Then** the server returns `401` RFC 7807 before any board-specific logic runs.

**Given** a user's role is changed from Editor to Viewer (STORY-1.3) **When** they subsequently call a mutating endpoint **Then** the server immediately enforces their new Viewer restriction (role is re-read from the database per request — no in-memory caching of role).

**Given** a user is removed from a board (STORY-1.4) **When** they subsequently call any endpoint for that board **Then** the server returns `404` (non-member) for all subsequent requests.

**Given** an authorization check determines the caller's role **When** the board id in the route does not match any board the caller is a member of **Then** the check returns 404 (not 403), preventing membership enumeration.

**Given** the role×action matrix is implemented **When** a new mutation endpoint is added in any future epic **Then** it MUST invoke the same `IBoardAuthorizationService` (or equivalent policy) — not implement its own ad-hoc role check. The service interface and its contract are defined in this story.

## NFR / constraints

- **Security / authorization (the spine):** This story defines the canonical authorization model. All epics EPIC-2 through EPIC-9 that introduce mutations MUST cite this convention and invoke the shared authorization service — they do not re-define it. The `403` / `404` / `401` status code meanings are frozen here and must not be varied in later epics.
- **Implementation:** Authorization logic lives in the **Application layer** as `IBoardAuthorizationService` (interface in Application; implementation in Infrastructure). The Domain layer has no dependency on it. The Api layer applies it via a policy/filter or at the top of each command handler.
- **No role caching:** the caller's `BoardMembership` row is read from the database on each authorized request. No in-process role cache (avoids stale-role window after STORY-1.3/1.4).
- **Clean Architecture:** no EF / Microsoft.* types in Domain; the authorization service is resolved via DI, not instantiated inline.
- **RFC 7807:** every 401, 403, and 404 path returns a structured problem-detail response (`type`, `title`, `status`, `detail`).
- **Testability:** the authorization service MUST be unit-testable in isolation (injectable, mockable) so that handler tests can assert role enforcement without a database.

## Out of scope

- Row-level security at the database layer (SQL Server RLS) — server-side application-layer enforcement is sufficient for v1.
- Fine-grained permissions within a role (e.g. an Editor who can comment but not delete cards) — three-tier role matrix only.
- Caching of membership for performance — deferred behind `ICacheService` seam (§2.11) if needed.
- The SignalR hub authorization (EPIC-6 will call the same `IBoardAuthorizationService` when clients subscribe to board groups).

**Traces:** §1.4, §1.6, §2.2; EPIC-4 (card metadata row); EPIC-5 (comment authority rows)
**Tests:** `BoardAuthorizationServiceTests` (Application unit — every cell in the role×action matrix; non-member → NotFound exception; unauthenticated → Unauthenticated exception; role-change takes effect immediately; Editor-deletes-own-comment → allowed; Editor-deletes-other-comment → 403; Owner-deletes-any-comment → allowed; Viewer-adds-comment → 403; card-metadata mutations: Editor → allowed, Viewer → 403); `ViewerMutationForbiddenTests` (Api integration — representative mutating endpoints return 403 for a Viewer token, including comment create and card-metadata endpoints); `EditorOwnerOnlyForbiddenTests` (Api integration — member-management and board-delete endpoints return 403 for an Editor token; Editor deleting another user's comment returns 403); `NonMemberEndpointTests` (Api integration — all board endpoints return 404 for a non-member authenticated token)
