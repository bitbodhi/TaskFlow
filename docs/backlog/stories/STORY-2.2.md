# STORY-2.2: Rename a list

**Epic:** EPIC-2

As an Editor or Owner of a board, I want to rename an existing list, so that I can keep column labels accurate as the project evolves.

## Acceptance Criteria

**Given** I am an Owner or Editor of a board **When** I PATCH/PUT a list's name with a valid value (1–100 characters) **Then** the list is updated and persisted, and I receive `200 OK` with the updated list DTO (id, boardId, name, position, updatedAt).

**Given** I submit a blank name or one longer than 100 characters **When** I rename a list **Then** I receive an RFC 7807 `400` validation problem and the list name is unchanged.

**Given** I am an unauthenticated caller **When** I attempt to rename a list **Then** I receive `401` and the list is unchanged.

**Given** I am authenticated but not a member of the board **When** I attempt to rename a list on that board **Then** I receive `404` (existence not disclosed to non-members) and the list is unchanged.

**Given** I am a Viewer of the board **When** I attempt to rename a list **Then** I receive `403` (member whose role forbids the mutation) and the list is unchanged.

**Given** the list id in the URL does not exist (or belongs to a different board) **When** I attempt to rename it **Then** I receive `404` and no change occurs.

**Given** a list is successfully renamed **When** another board member views the board **Then** real-time propagation < 1 s (via EPIC-6) delivers the updated name to connected clients.

## NFR / constraints

- **Security / authorization:** rename is permitted for Owner and Editor only. Status-code convention (per EPIC-1 / STORY-1.5): `401` unauthenticated; `404` non-member or list not found; `403` Viewer; `400` validation.
- **Idempotency:** renaming to the current name is allowed and returns `200` without writing to the DB (the Application layer should short-circuit on no-op). This is a quality-of-life decision — clients that optimistically confirm before the server responds may retry.
- **Data:** list name 1–100 characters, same constraint as creation.
- **API:** DTOs only; RFC 7807 on every error; `updatedAt` returned to allow clients to reconcile concurrent edits.
- **Clean Architecture (§2.2):** rename use-case in Application; no Domain logic beyond the invariant; EF Core update in Infrastructure; thin controller in Api.
- **Real-time:** propagation constraint noted; EPIC-6 owns the SignalR story.

## UX spec (skeleton)

List column header shows the list name as an **editable label**: double-click (or single-click on an edit icon) converts it to an inline text input pre-filled with the current name (auto-selected). **Save** on Enter or blur; **Cancel** on `Esc` (reverts). If saving, the field enters a submitting state (disabled). On success the header reverts to a label with the new name. States: view / editing / submitting / validation error (inline `aria-describedby` tooltip) / server-error banner (`role="alert"`). WCAG 2.2 AA: the edit trigger is reachable by keyboard; announced rename completion via `aria-live`. No external comp — this note is the design reference.

## Out of scope

- Bulk rename; list colour/icon changes.
- Conflict resolution for simultaneous renames (last-write-wins via `updatedAt` is sufficient in v1).
- Real-time delivery story (EPIC-6).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.5, §1.6, §2.2
**Tests:** `RenameListHandlerTests` (Application unit — success, no-op short-circuit, invalid-name, authz denied Viewer, authz denied non-member, list not found); `RenameListEndpointTests` (Api integration — 200 DTO with updatedAt, 400 RFC 7807, 401, 403 Viewer, 404 non-member, 404 wrong-list-id); `ListNameEditor.spec.ts` (Vue/Vitest — double-click to edit, Enter/blur saves, Esc reverts, validation error, a11y)
