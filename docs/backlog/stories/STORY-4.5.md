# STORY-4.5: Set / clear a card priority

**Epic:** EPIC-4

As a board Editor or Owner, I want to set or clear a priority level on a card, so that my team can immediately see which work is most urgent.

## Acceptance Criteria

**Given** I am an Editor or Owner **When** I PATCH `/boards/{boardId}/cards/{cardId}` with `{ "priority": "High" }` (or `"Medium"`, `"Low"`, `"None"`) **Then** the priority is stored, I receive `200 OK` with the updated card DTO (including `"priority": "High"`), and the priority badge is visible on the card; real-time propagation <1s (via EPIC-6).

**Given** a card has a priority set **When** I PATCH with `{ "priority": "None" }` **Then** the priority is cleared to `None`, I receive `200 OK` with `"priority": "None"`, and the priority badge is HIDDEN; real-time propagation <1s (via EPIC-6).

**Given** I submit a `priority` value outside the allowed enum (`None`, `Low`, `Medium`, `High`) **When** I PATCH the card **Then** I receive RFC 7807 `400` ("Invalid priority value. Allowed values: None, Low, Medium, High") and no change is made.

**Given** a card has not had its priority set (new cards), or priority is explicitly set to `None` **When** a member views the card **Then** the priority field reads `None` (the default); the priority badge is HIDDEN (consistent with "No labels"/"No assignees" empty states — the badge is not shown as "None").

**Given** I am a Viewer of the board **When** I attempt to set or clear a priority **Then** I receive `403 Forbidden` and no change is made.

**Given** I am not a member of the board or present no/invalid token **When** I attempt to set or clear a priority **Then** I receive `404` (non-member) or `401` (unauthenticated) and no change is made.

**Given** the `cardId` in the PATCH request does not exist or does not belong to `boardId` **When** I attempt to set or clear a priority **Then** I receive `404` (generic — existence not disclosed) and no change is made.

**Given** the card detail panel is open **When** it is loading or encounters an error **Then** appropriate loading and error states are shown; the priority selector meets WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** set priority requires Editor or Owner role, enforced server-side. Viewer → `403`. Non-member / unauthenticated → `404` / `401`.
- **cardId scope:** a PATCH referencing a `cardId` that does not exist or does not belong to `boardId` returns `404` (generic — existence not disclosed, matching STORY-0.4's cross-board guard); no data is modified.
- **None display:** when priority is `None`, the priority badge is HIDDEN — no "None" label or placeholder chip is rendered; this matches the empty-state convention of "No labels" / "No assignees" across the card detail panel.
- **Priority enum (canonical):** `None` (0) / `Low` (1) / `Medium` (2) / `High` (3) — defined as an enum in Domain; stored as `int` column in SQL Server; serialised as the string name in API DTOs (e.g. `"High"`). Default is `None`. The enum is the single source of truth — no magic strings outside Domain.
- **`None` is explicit:** `None` is a first-class value meaning "no priority set"; it is the default for new cards and the value used to clear priority (not `null`).
- **Partial update:** shares the same PATCH endpoint as STORY-4.4 (`dueDate`); fields are independent; omitting a field leaves it unchanged.
- **API:** DTOs only at the boundary; enum serialised as string name; RFC 7807 on every error path.
- **Clean Architecture:** `Priority` enum in Domain; `Card.Priority` property in Domain entity; update use-case in Application; EF Core stores as `int`; thin controller in Api. No EF/Microsoft.* in Domain.
- **Real-time:** priority changes propagate to all board members <1s (via EPIC-6); this story defines the data contract only.
- **Accessibility:** priority indicator must not rely on color alone — the text label ("High", "Medium", "Low") must always be visible alongside any color/icon. The selector is a labelled control (not an unlabelled color-only picker).

## UX spec (skeleton)
Card detail panel → **Priority** section. Displays current priority as a labelled badge: text ("High" / "Medium" / "Low") plus a distinguishing color or icon (color is supplementary, not sole indicator). **Select priority:** a dropdown/select with four options — None, Low, Medium, High — each shown as text (with optional color). Selecting an option PATCHes the card immediately (optimistic update). When priority is `None` (default or explicitly cleared), the badge is HIDDEN — no chip or "None" label is rendered; the section shows only the selector control, consistent with "No labels"/"No assignees" empty states elsewhere in the card detail. States: loading (skeleton); default/None state (badge hidden, selector visible); error banner (`role="alert"`). The `<select>` (or custom combobox) has a visible `<label>` ("Priority"); keyboard-navigable.

## Out of scope
- Custom priority levels or additional enum values.
- Filtering / sorting cards by priority (EPIC-8).
- Automated escalation based on priority.

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.6, §2.2
**Tests:** `SetCardPriorityHandlerTests` (Application unit — set each valid enum value, set None to clear, invalid string → 400, Viewer → 403, non-member → 404, cardId not found → 404, cardId from different board → 404); `CardPriorityEndpointTests` (Api integration — 200 for each valid priority, 400 invalid value, 401, 403, 404 non-member, 404 wrong cardId, enum round-trips as string in JSON, default None on newly created card); `CardPrioritySection.spec.ts` (Vue/Vitest — select each priority, clear to None hides badge entirely, badge hidden when priority is None, badge visible for Low/Medium/High, loading/error states, a11y: text label always visible not color-only, `<select>` has visible label)
