# STORY-3.2: Edit a card (title and description)

**Epic:** EPIC-3

As an Editor or Owner of a board, I want to edit a card's title and description, so that I can keep the task information accurate as work evolves.

## Acceptance Criteria

**Given** I am an authenticated Editor or Owner of the board **When** I PATCH `/cards/{cardId}` with a valid `title` (1–255 chars) and/or an optional `description` (0–4000 chars, or explicit `null` to clear it) **Then** the card is updated in the database, I receive `200 OK` with the full updated card DTO (`id`, `title`, `description`, `listId`, `boardId`, `updatedAt`), and `updatedAt` is refreshed.

**Given** I am an Editor or Owner with unsaved edits in the card edit form **When** I click the explicit **Save** button **Then** client-side validation fires, and on passing, the PATCH is submitted; on Cancel the form closes and unsaved edits are discarded.

**Given** I submit a blank title, a title longer than 255 characters, or a description longer than 4000 characters **When** I PATCH the card **Then** I receive `400` RFC 7807 with field-level validation errors and the card is unchanged.

**Given** I am a Viewer on the board **When** I PATCH `/cards/{cardId}` **Then** I receive `403` RFC 7807 and the card is unchanged.

**Given** I am not a member of the board, or present no/invalid JWT **When** I PATCH `/cards/{cardId}` **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated) and the card is unchanged.

**Given** I am an Editor or Owner editing a card **When** I submit the edit form in the UI **Then** the card detail shows the updated title and description immediately after save, with a loading/submitting state during the request and an inline error banner if the request fails.

**Given** the edit form renders **When** viewed by a Viewer **Then** all inputs are disabled or the edit controls are hidden — Viewer cannot initiate an edit in the UI.

**Given** the edit form is displayed **When** rendered **Then** it meets WCAG 2.2 AA: labels associated with inputs, inline validation errors announced via `aria-live`, visible focus indicators.

## NFR / constraints
- **Authorization:** Editor and Owner may edit; Viewer receives `403`. Checked server-side on every request.
- **Data bounds:** title 1–255 chars (consistent with EPIC-0 domain entity and EF column constraint); description 0–4000 chars (nullable; `null` in the PATCH payload explicitly clears the field). Both enforced in Domain, EF column, and client validation.
- **Partial update:** PATCH semantics — only fields present in the payload are updated; omitting `title` retains the existing title; omitting `description` retains the existing description; sending `"description": null` clears it.
- **Concurrency:** `updatedAt` is updated on every successful write. No optimistic-concurrency ETag required in this story (deferred); last-write-wins is acceptable.
- **Real-time:** other board members' UIs will not reflect this edit in real time until EPIC-6 is in place — note as a constraint, not authored here.
- DTOs only at the API boundary; RFC 7807 on all error paths; structured logging of edit events.

## UX spec (skeleton)
In the card detail panel: **title** renders as an editable inline field (click-to-edit or always-editable input, max 255 chars, required). **Description** renders as a textarea (max 4000 chars, placeholder "Add a description…"). An explicit **Save** button commits the PATCH on user intent; a **Cancel** button discards unsaved edits. Validation fires on Save, not on blur. States: **idle** (inputs enabled for Editor/Owner; read-only display for Viewer), **submitting** (Save button disabled + spinner), **validation error** (inline message below offending field, `aria-describedby`), **server error** (alert banner `role="alert"` with dismiss + retry). Character counters shown at-threshold (e.g. when within 20 chars of limit). Destructive clearing of description is not considered destructive enough to require a confirm dialog.

## Out of scope
- Moving or reordering the card (STORY-3.4, STORY-3.5).
- Card metadata fields: labels, assignees, due dates, priorities (EPIC-4).
- Optimistic-concurrency conflict detection (deferred).
- Rich-text / markdown rendering in the description (deferred — plain text only in v1).
- Auto-save on blur (deferred — v1 uses explicit Save only).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `EditCardCommandHandlerTests` (Application unit — success full/partial patch, title blank → invalid, title >255 → invalid, description >4000 → invalid, description null clears field, Viewer role → authz exception, non-member → not-found); `EditCardEndpointTests` (Api integration — 200 updated DTO, 400 field errors RFC 7807, 401 no token, 403 Viewer, 404 non-member, persistence verified on read-back); `CardEditForm.spec.ts` (Vitest — partial patch logic, character counter, submitting state, validation fires on Save not on blur, Cancel discards unsaved edits, Viewer sees read-only, a11y)
