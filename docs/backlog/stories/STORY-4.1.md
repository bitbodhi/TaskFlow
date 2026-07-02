# STORY-4.1: Manage a board's label set (create / edit / delete)

**Epic:** EPIC-4

As a board Editor or Owner, I want to create, rename, recolour, and delete labels scoped to my board, so that my team has a consistent, reusable set of categories to apply to cards.

## Acceptance Criteria

**Given** I am an Editor or Owner of a board **When** I POST `/boards/{boardId}/labels` with a valid `name` (1–50 characters) and a valid `color` (CSS hex, e.g. `#FF5733`) **Then** the label is created, persisted, and I receive `201 Created` with a label DTO (`id`, `boardId`, `name`, `color`).

**Given** I am an Editor or Owner **When** I PATCH `/boards/{boardId}/labels/{labelId}` with an updated `name` and/or `color` **Then** the label is updated and I receive `200 OK` with the updated DTO; real-time propagation <1s (via EPIC-6).

**Given** I am an Editor or Owner **When** I DELETE `/boards/{boardId}/labels/{labelId}` **Then** the label is removed from the board AND all card–label associations for that label are cascade-removed atomically; I receive `204 No Content`; real-time propagation <1s (via EPIC-6).

**Given** a board has existing labels **When** I GET `/boards/{boardId}/labels` **Then** I receive a paginated list (`{ items, page, pageSize, total }`) of all labels for that board.

**Given** I submit a `name` that already exists on the same board (case-insensitive) **When** I create or rename a label **Then** I receive RFC 7807 `400` and no change is made.

**Given** I PATCH a label with a new `name` value that already exists on the same board (case-insensitive) **When** the update is processed **Then** I receive RFC 7807 `400` and no change is made; a color-only PATCH (no `name` supplied) does NOT re-validate the existing name and succeeds if the color is valid.

**Given** the `labelId` in a PATCH or DELETE request does not exist, or exists but does not belong to `boardId` **When** I attempt the operation **Then** I receive `404` (generic — existence not disclosed) and no change is made.

**Given** I submit a `name` that is blank or longer than 50 characters, or a `color` that is not a valid CSS hex string **When** I create or rename/recolour a label **Then** I receive RFC 7807 `400` and no change is made.

**Given** I am a Viewer of the board **When** I attempt to create, update, or delete a label **Then** I receive `403 Forbidden` and no change is made.

**Given** I am not a member of the board or present no/invalid token **When** I attempt any label mutation or read **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated); no data is returned.

**Given** I am an Editor or Owner viewing the label management panel **When** it is loading, empty, or encounters an error **Then** appropriate loading, empty ("No labels yet — create one to get started"), and error states are shown; all controls meet WCAG 2.2 AA.

**Given** I initiate a label delete **When** the confirmation dialog appears **Then** I must confirm explicitly before the label is deleted; cancelling leaves the label intact.

## NFR / constraints
- **Security / authorization:** all mutations require Editor or Owner role, enforced server-side. Viewer → `403`. Non-member / unauthenticated → `404` / `401`.
- **Data:** `name` 1–50 characters (enforced in Domain entity, EF column, and client validation). `color` must be a valid 6-digit or 3-digit CSS hex string (e.g. `#FF5733`, `#F53`); non-hex strings such as `red` or `rgb(...)` are rejected with `400`. `name` is unique per board (case-insensitive, at the database constraint and application layer).
- **Partial-update name uniqueness:** the duplicate-name check is only triggered when a new `name` value is present in the PATCH body; a color-only PATCH does not re-evaluate the (unchanged) existing name.
- **labelId scope:** a PATCH or DELETE referencing a `labelId` that does not exist or does not belong to `boardId` returns `404` (generic — existence not disclosed); no data is modified.
- **Cascade delete:** deleting a label removes all `CardLabel` join-table rows for that label in the same DB transaction; no orphans.
- **Pagination:** `GET /boards/{boardId}/labels` returns `{ items, page, pageSize, total }` (offset-based, page default 1, pageSize default 20 max 100).
- **API:** DTOs only at the boundary; RFC 7807 on every error path.
- **Clean Architecture:** `Label` entity and `CardLabel` join in Domain; use-cases (CreateLabel, UpdateLabel, DeleteLabel, ListLabels) in Application; EF Core persistence in Infrastructure; thin controller in Api. No EF/Microsoft.* references in Domain.
- **a11y:** label color is never the ONLY distinguisher — the `name` field is always required and displayed alongside (or instead of) the color swatch in all UI surfaces.
- **Real-time:** update and delete events propagate to all board members <1s (via EPIC-6); this story defines the data contract; SignalR wiring is EPIC-6's responsibility.

## UX spec (skeleton)
Board settings panel → **Labels** tab. List of existing labels displayed as rows: colored swatch + name + Edit / Delete icon buttons. **Create label** form (inline or modal): `name` text input (required, max 50 chars) + `color` hex picker/input (required) + Save / Cancel. **Edit:** clicking Edit populates the same form inline. **Delete:** icon triggers a confirmation dialog ("Delete label 'Bug'? It will be removed from all cards.") with Confirm / Cancel. States: loading spinner; empty state ("No labels yet"); inline validation errors (`aria-describedby`); server-error banner (`role="alert"`). Keyboard: all controls keyboard-accessible; focus returns to the triggering element after modal close; color input accepts typed hex. Color swatch is decorative (`aria-hidden`); name is the primary label identifier in all contexts.

## Out of scope
- Applying labels to cards (STORY-4.2).
- Filtering cards by label (EPIC-8).
- Label ordering / drag-and-drop.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.5, §1.6, §2.2
**Tests:** `CreateLabelHandlerTests`, `UpdateLabelHandlerTests`, `DeleteLabelHandlerTests`, `ListLabelsHandlerTests` (Application unit — success, duplicate name on create, duplicate name on rename, color-only PATCH does not re-check name uniqueness, invalid name/color including non-hex values `red`/`rgb(...)`, 3-digit valid hex `#F53` accepted, 6-digit valid hex `#FF5733` accepted, authz denied for Viewer, cascade delete removes card associations, PATCH/DELETE with unknown labelId → not-found, PATCH/DELETE with labelId from different board → not-found); `LabelEndpointTests` (Api integration — 201/200/204/400/401/403/404, PATCH color-only succeeds without name conflict, PATCH unknown labelId → 404, DELETE unknown labelId → 404, cascade verified via card-label query after delete, paginated list shape); `LabelManagementPanel.spec.ts` (Vue/Vitest — form validation, empty/loading/error states, delete confirmation dialog, a11y including color-not-only-distinguisher)
