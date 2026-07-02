# STORY-4.4: Set / clear a card due date

**Epic:** EPIC-4

As a board Editor or Owner, I want to set or clear a due date on a card, so that my team knows when work is expected to be complete.

## Acceptance Criteria

**Given** I am an Editor or Owner **When** I PATCH `/boards/{boardId}/cards/{cardId}` with `{ "dueDate": "<ISO 8601 UTC datetime>" }` **Then** the due date is stored as UTC, I receive `200 OK` with the updated card DTO (including `dueDate` as ISO 8601 UTC string), and the due date is visible on the card; real-time propagation <1s (via EPIC-6).

**Given** a card has a due date **When** I PATCH with `{ "dueDate": null }` **Then** the due date is cleared, I receive `200 OK` with `"dueDate": null`, and the due date indicator is removed from the card; real-time propagation <1s (via EPIC-6).

**Given** I submit a `dueDate` value that is not a valid ISO 8601 datetime string **When** I PATCH the card **Then** I receive RFC 7807 `400` and no change is made.

**Given** I am a Viewer of the board **When** I attempt to set or clear a due date **Then** I receive `403 Forbidden` and no change is made.

**Given** I am not a member of the board or present no/invalid token **When** I attempt to set or clear a due date **Then** I receive `404` (non-member) or `401` (unauthenticated) and no change is made.

**Given** the `cardId` in the PATCH request does not exist or does not belong to `boardId` **When** I attempt to set or clear a due date **Then** I receive `404` (generic — existence not disclosed) and no change is made.

**Given** a card has a due date in the past **When** a member views the card **Then** the due date is displayed with an overdue indicator (e.g. red color + "Overdue" label text); overdue calculation is done client-side using the viewer's local time; the server stores and returns UTC only.

**Given** the card detail panel is open **When** it is loading or encounters an error **Then** appropriate loading and error states are shown; the date picker meets WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** set and clear due date require Editor or Owner role, enforced server-side. Viewer → `403`. Non-member / unauthenticated → `404` / `401`.
- **cardId scope:** a PATCH referencing a `cardId` that does not exist or does not belong to `boardId` returns `404` (generic — existence not disclosed, matching STORY-0.4's cross-board guard); no data is modified.
- **Timezone:** `dueDate` is stored as `DateTimeOffset` (UTC) in SQL Server; API always accepts and returns ISO 8601 UTC strings (e.g. `"2026-08-01T17:00:00Z"`). The server has no concept of "overdue" — that is a client display concern.
- **Null is valid:** `dueDate` may be null (not set). A PATCH with `"dueDate": null` explicitly clears it.
- **Partial update:** the PATCH endpoint for a card accepts any subset of patchable fields (`dueDate`, and those added by STORY-4.5); omitting a field leaves it unchanged.
- **API:** DTOs only at the boundary; RFC 7807 on every error path.
- **Clean Architecture:** `dueDate` field on `Card` entity in Domain; update use-case in Application; EF Core in Infrastructure. No EF/Microsoft.* in Domain.
- **Real-time:** due-date changes propagate to all board members <1s (via EPIC-6); this story defines the data contract only.
- **Accessibility:** the date picker (or date input) must be keyboard-accessible and have a visible label. The overdue indicator must not rely on color alone — the text "Overdue" (or equivalent) must accompany the color change.

## UX spec (skeleton)
Card detail panel → **Due date** section. Displays current due date formatted in the user's locale (derived from the UTC value). If overdue, shows a colored badge AND the text "Overdue" (color is not the sole indicator). **Set:** a date-time picker (or `<input type="datetime-local">`) allows selection; saving PATCHes the card. **Clear:** a "Remove due date" button (visible when a due date is set) sets `dueDate: null`. States: loading (skeleton); no-due-date state ("No due date"); overdue state (badge + text); error banner (`role="alert"`). All inputs have visible `<label>` elements; date picker is keyboard-accessible.

## Out of scope
- Server-side overdue computation or notifications.
- Recurring due dates.
- Filtering cards by due date (EPIC-8).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.6, §2.2
**Tests:** `SetCardDueDateHandlerTests` (Application unit — set valid UTC, clear with null, invalid date string → 400, Viewer → 403, cardId not found → 404, cardId from different board → 404); `CardDueDateEndpointTests` (Api integration — 200 with dueDate, 200 with null dueDate, 400 invalid format, 401, 403, 404 non-member, 404 wrong cardId, UTC round-trip verified); `CardDueDateSection.spec.ts` (Vue/Vitest — set date, clear date, overdue indicator shows text not color-only, loading/error states, a11y: label on input, keyboard navigation)
