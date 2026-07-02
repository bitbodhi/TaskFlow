# STORY-0.4: Add a card to a board (persists)

**Epic:** EPIC-0

As an authenticated board Owner, I want to add a card with a title to my board, so that I can capture a task and trust it is still there when I come back.

## Acceptance Criteria

**Given** I am the Owner of a board (with its seeded default list) **When** I POST a card with a valid title (**1–255 characters**) **Then** the card is created in that list, persisted, and I receive `201 Created` with a card DTO (id, title, listId).

**Given** I am authenticated **When** I POST a card whose `listId` does not belong to a board I am a member of (or does not exist) **Then** I receive `404` RFC 7807 (the list's existence is not disclosed to non-members) and no card is created — list ownership is validated server-side, not assumed from board membership alone.

**Given** I submit a blank title or one longer than **255 characters** **When** I add a card **Then** I receive an RFC 7807 `400` validation problem and no card is created.

**Given** I am not a member of the board, or present no/invalid token **When** I POST a card **Then** I receive `404` (non-member — existence not disclosed) or `401` (unauthenticated) and no card is created (server-side authorization — **Owner only** in EPIC-0).

**Given** I created a card **When** I reload the board — or sign out and back in and re-open it — **Then** `GET` of the board's cards returns the card exactly as created (the epic's headline persistence proof), as a **paginated** response.

**Given** the board view **When** it has no cards yet, is loading, or errors **Then** empty, loading, and error states are shown, and the add-card form meets WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** card creation and the card-read are **authenticated and authorized server-side** against board membership. In EPIC-0 the **only role is Owner** (the board's creator) — Owner may add; non-member cannot. The **Editor** role (and any other non-Owner authorization) is deferred to the later epic that introduces membership management, and is intentionally not wired here (no dead role-check code). **Status codes** follow the shared convention defined in STORY-0.3: `401` unauthenticated, `404` non-member (existence not disclosed), `400` validation; `403` is not reachable in EPIC-0. The target **`listId` is verified to belong to the caller's board** before the card is created (prevents cross-board insertion / IDOR).
- **Data:** card title is **1–255 characters** (enforced consistently in the Domain entity, EF column, and client validation).
- **Performance / pagination contract:** the board's card list is **paginated** (§1.6 — board target ~200 cards loads in <1.5s), using the **same envelope as STORY-0.3**: offset-based, `page` (1-indexed, default 1) + `pageSize` (default 20, max 100); response `{ items: [...], page, pageSize, total }`.
- **API:** **DTOs only** at the boundary; input validated; **RFC 7807** on every error path.
- **Clean Architecture (§2.2):** `Card` entity in Domain; add-card use-case in Application; EF Core persistence in Infrastructure; thin controller in Api.
- **Reliability:** EF Core migration creates `Cards`; structured logging; global 7807 error handling.
- **Accessibility:** WCAG 2.2 AA on the add-card form and board view.

## UX spec (skeleton)
Board view showing the default list as a column, with an **empty state** ("No cards yet") and an add-card control: a single **title** input (1–255 chars) + submit (or Enter). Cards render as a simple stacked list below. States: empty / loading / submitting / validation error (inline, `aria-describedby`) / server-error banner (`role="alert"`). Keyboard: add-card control focusable, field auto-focused when opened, Enter submits, `Esc` cancels. No external comp — this note is the design reference.

## Out of scope
- Editing, moving, or deleting cards; drag-and-drop ordering / card positions; labels, assignees, due dates, priorities, comments; **real-time propagation to other users** (no SignalR in EPIC-0); the activity feed.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.1, §1.3, §1.6, §2.2
**Tests:** `AddCardHandlerTests` (Application unit — success, invalid-title incl. >255 chars, authz-denied, `listId` belonging to another board → rejected); `AddCardEndpointTests` (Api integration — 201 DTO, 400 RFC 7807, 401 unauthenticated, 404 non-member + foreign-`listId`, persistence read-back after re-auth); `BoardCardsEndpointTests` (Api integration — paginated shape); `AddCardForm.spec.ts` (Vue/Vitest — validation, empty/loading/error states, a11y)
