# STORY-0.3: Create a board

**Epic:** EPIC-0

As an authenticated user, I want to create a board with a name, so that I have a place to organize my cards and can return to it later.

## Acceptance Criteria

**Given** I am authenticated **When** I POST a board with a valid name (**1–100 characters**) **Then** the board is created and persisted with me recorded as its **Owner**, a single default list is seeded inside it, and I receive `201 Created` with a board DTO (id, name, my role = Owner).

**Given** board creation is in progress **When** seeding the default list fails **Then** the whole operation is rolled back in a **single transaction** — no orphaned board exists without its default list.

**Given** I am authenticated **When** I submit a blank name or one longer than **100 characters** **Then** I receive an RFC 7807 `400` validation problem and no board is created.

**Given** I present no token or an invalid JWT **When** I POST a board **Then** I receive `401` and no board is created.

**Given** a board I own **When** I `GET /boards/{id}` (e.g. after a page reload) **Then** I retrieve the board; **Given** a board I am not a member of **When** I GET it **Then** I receive `404` — its existence is not disclosed to non-members (server-side authorization on read).

**Given** I have signed back in **When** I `GET /boards` **Then** I receive my boards as a **paginated** response (page metadata present even for a single board), so the SPA can land me back on my board.

**Given** the create-board UI **When** there are no boards yet, while loading, or on error **Then** empty, loading, and error states are shown, and the form/dialog meets WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** every board mutation and read is **authenticated and authorized server-side** — the creator becomes Owner; non-members cannot read or mutate the board. **Status-code convention (shared across this epic):** `401` unauthenticated; `404` for an authenticated **non-member** (the resource's existence is not disclosed — prevents enumeration); `403` is reserved for a *member* whose role forbids the action (not reachable in EPIC-0, where the only role is Owner); `400` for validation.
- **Performance / pagination contract:** `GET /boards` is **paginated** (§1.6 — all list endpoints paginated), even though EPIC-0 expects ~1 board. Contract (shared by every list endpoint in this epic): **offset-based**, query params `page` (1-indexed, default 1) and `pageSize` (default 20, max 100); response envelope `{ items: [...], page, pageSize, total }`.
- **API:** **DTOs only** at the boundary; input validated; **RFC 7807** on every error path.
- **Data:** board name is **1–100 characters** (enforced in the Domain entity, the EF column, and client validation consistently); board creation + default-list seeding commit in **one transaction**.
- **Clean Architecture (§2.2):** `Board`, `List`, `BoardMember` entities in Domain; create-board use-case in Application; EF Core persistence in Infrastructure; thin controller in Api.
- **Reliability:** EF Core migration creates `Boards`, `Lists`, `BoardMembers`; structured logging; global 7807 error handling.
- **Accessibility:** WCAG 2.2 AA on the create-board form and the boards view.

## UX spec (skeleton)
Boards view with an **empty state** ("No boards yet — create your first") and a create-board control (inline field or small dialog) with a single **name** input (1–100 chars) + submit. States: empty / loading (skeleton or spinner) / submitting / validation error (inline, `aria-describedby`) / server-error banner (`role="alert"`). On success, navigate straight into the new board. Keyboard: control is focusable, field auto-focused when opened, `Esc` closes a dialog. No external comp — this note is the design reference.

## Out of scope
- Renaming/deleting boards, inviting/managing members, assigning roles beyond creator-as-Owner, multiple lists or any list UI/management, real-time propagation of board creation to other users.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.3, §1.4, §1.6, §2.2
**Tests:** `CreateBoardHandlerTests` (Application unit — success seeds default list + Owner membership, invalid-name incl. >100 chars, list-seed failure rolls back the board); `CreateBoardEndpointTests` (Api integration — 201 DTO, 400 RFC 7807, 401 unauthenticated, GET-by-id authz 200 owner vs 404 non-member); `ListBoardsEndpointTests` (Api integration — paginated shape); `BoardCreateForm.spec.ts` (Vue/Vitest — validation, empty/loading/error states, a11y)
