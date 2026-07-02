# STORY-2.1: Create a list on a board

**Epic:** EPIC-2

As an Editor or Owner of a board, I want to add a named list to the board, so that I can create new columns to organize cards.

## Acceptance Criteria

**Given** I am an Owner or Editor of a board **When** I POST a list with a valid name (1–100 characters) **Then** the list is created, appended after the last existing list on the board, assigned a stable fractional-index rank (see NFR — Position model), persisted, and I receive `201 Created` with a list DTO (id, boardId, name, rank/position, createdAt). The POST body accepts `name` only; no position parameter is accepted (repositioning after creation is handled by STORY-2.4).

**Given** I submit a blank name or one longer than 100 characters **When** I create a list **Then** I receive an RFC 7807 `400` validation problem and no list is created.

**Given** I am an unauthenticated caller **When** I POST a list **Then** I receive `401` and no list is created.

**Given** I am authenticated but not a member of the board **When** I POST a list **Then** I receive `404` (existence not disclosed to non-members) and no list is created.

**Given** I am a Viewer of the board **When** I POST a list **Then** I receive `403` (member whose role forbids the mutation) and no list is created.

**Given** I create a list **When** another board member views the board **Then** real-time propagation < 1 s (via EPIC-6) delivers the new list to connected clients.

**Given** a board already has 500 lists **When** I POST a new list **Then** I receive `400` RFC 7807 ("list limit reached") and no list is created.

## NFR / constraints

- **Security / authorization:** creation is permitted for Owner and Editor only. Status-code convention (per EPIC-1 / STORY-1.5): `401` unauthenticated; `404` non-member; `403` Viewer; `400` validation.
- **Position model:** lists carry a `double`-precision `Position` column (fractional index; the rank column is named `Position` in the EF model). New lists are always **appended**: `rank = max(existing) + 1.0` (or `1.0` for the first list). There is no "insert at position" in v1 — repositioning uses STORY-2.4. When the gap between neighbours falls below `1e-9`, a background re-normalisation writes integer ranks (1, 2, 3 …) for the board's lists in one transaction — callers never see an error.
- **Data:** list name is 1–100 characters (Domain entity, EF column, client validation). A board may have up to 500 lists (Domain invariant, enforced in Application; returns `400` "list limit reached" if exceeded).
- **API:** DTOs only at the boundary; RFC 7807 on every error path.
- **Clean Architecture (§2.2):** `List` entity in Domain (id, boardId, name, position, createdAt); create-list use-case in Application; EF Core persistence in Infrastructure; thin controller in Api.
- **Real-time:** propagation constraint noted; EPIC-6 owns the SignalR story.

## UX spec (skeleton)

Board view showing existing lists as columns. An **"Add list"** button appears after the last list and in the empty-board state ("No lists yet — add your first"). Clicking opens an inline form: **Name** input (1–100 chars, auto-focused) + **Add** button (or Enter). Pressing `Esc` cancels. The new list column appears immediately on `201`. States: idle / submitting (button disabled) / validation error (inline, `aria-describedby`) / server-error banner (`role="alert"`). WCAG 2.2 AA: button and input are keyboard-reachable; error messages are announced via `aria-live`. No external comp — this note is the design reference.

## Out of scope

- Moving cards into the new list (EPIC-3).
- Setting list colour/icon; archiving.
- Real-time delivery story (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.5, §1.6, §2.2
**Tests:** `CreateListHandlerTests` (Application unit — success appends rank, invalid-name blank → 400, invalid-name >100 chars → 400, board-at-500-lists → 400 "list limit reached", authz denied for Viewer, authz denied for non-member); `CreateListEndpointTests` (Api integration — 201 DTO with rank, 400 RFC 7807 validation, 400 RFC 7807 list-limit-reached, 401, 403 Viewer, 404 non-member); `AddListForm.spec.ts` (Vue/Vitest — validation, empty/loading/error states, a11y, Esc cancel)
