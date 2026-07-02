# STORY-4.3: Assign / unassign a board member to a card

**Epic:** EPIC-4

As a board Editor or Owner, I want to assign board members to a card and remove those assignments, so that ownership and responsibility are clear at a glance.

## Acceptance Criteria

**Given** I am an Editor or Owner and `userId` is a current member of the board **When** I POST `/boards/{boardId}/cards/{cardId}/assignees` with `{ "userId": "<id>" }` **Then** the user is assigned to the card, I receive `201 Created` with `{ cardId, userId, displayName, avatarUrl }`, and the assignment is visible on the card; real-time propagation <1s (via EPIC-6).

**Given** a user is already assigned to a card **When** I POST the same `userId` again **Then** I receive RFC 7807 `400` ("User already assigned to this card") and no duplicate assignment is created.

**Given** I attempt to assign a `userId` who is NOT a current member of the board **When** I POST the assignment **Then** I receive RFC 7807 `400` ("User is not a member of this board") and no assignment is created (prevents assigning non-members).

**Given** a user has been removed from the board after being assigned to a card **When** an Editor or Owner views the card **Then** existing assignments are preserved and displayed (with a visual warning, e.g. "(removed from board)"); **When** an Editor or Owner attempts to add a NEW assignment for a removed user **Then** they receive `400` as above.

**Given** I am an Editor or Owner and a user is assigned to a card **When** I DELETE `/boards/{boardId}/cards/{cardId}/assignees/{userId}` **Then** the assignment is removed, I receive `204 No Content`, and the user no longer appears as an assignee; real-time propagation <1s (via EPIC-6).

**Given** the `userId` in the DELETE path is not currently assigned to the card **When** I attempt to remove them **Then** I receive RFC 7807 `404` ("Assignee not found on this card") and no change is made.

**Given** I am a Viewer of the board **When** I attempt to add or remove an assignee **Then** I receive `403 Forbidden` and no change is made.

**Given** I am not a member of the board or present no/invalid token **When** I attempt any assignee mutation **Then** I receive `404` (non-member) or `401` (unauthenticated); no change is made.

**Given** the card detail panel is open **When** it is loading, has no assignees, or encounters an error **Then** appropriate loading, empty ("No assignees"), and error states are shown; all controls meet WCAG 2.2 AA.

## NFR / constraints
- **Security / authorization:** assign and unassign require Editor or Owner role, enforced server-side. Viewer → `403`. Non-member / unauthenticated → `404` / `401`.
- **Assignee pool guard:** only current board members may be assigned; validated server-side at assignment time (prevents IDOR and assignment of arbitrary users). Existing assignments for removed members are not auto-purged (historical record preserved; UI flags them).
- **API:** DTOs only at the boundary; RFC 7807 on every error path. `GET /boards/{boardId}/cards/{cardId}/assignees` returns the list of current assignees (non-paginated — card assignees are bounded by board membership size, itself a finite set).
- **Clean Architecture:** `CardAssignee` join entity in Domain; use-cases in Application; EF Core in Infrastructure. No EF/Microsoft.* in Domain.
- **Real-time:** add and remove events propagate <1s (via EPIC-6); this story defines the data contract only.
- **Accessibility:** assignee avatars are decorative (`aria-hidden`); display name is the accessible text. Remove button has `aria-label="Remove assignee <displayName>"`.

## UX spec (skeleton)
Card detail panel → **Assignees** section. Displays assigned members as a row of avatar + name chips. Removed-member chips carry a "(removed from board)" badge. **Add assignee:** "Assign member" (or `+`) opens a searchable dropdown listing current board members not yet assigned; selecting one applies immediately (optimistic update). **Remove:** each chip has an `×` button (`aria-label="Remove assignee <displayName>"`). States: loading (skeleton chips); empty ("No assignees — assign a member"); error banner (`role="alert"`). Keyboard: dropdown is focusable and navigable; `Esc` closes; focus returns to trigger after close.

## Out of scope
- Managing board membership / inviting members (EPIC-1).
- "My Tasks" aggregated view across boards (EPIC-9).
- Notifying assignees by email or push (future).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.3, §1.4, §1.6, §2.2
**Tests:** `AssignCardMemberHandlerTests`, `UnassignCardMemberHandlerTests` (Application unit — success, duplicate assign → 400, non-member user → 400, unassign not-found → 404, Viewer → 403); `CardAssigneeEndpointTests` (Api integration — 201/204/400 duplicate/400 non-member/401/403/404, removed-member assignment preserved after board removal); `CardAssigneesSection.spec.ts` (Vue/Vitest — assign/unassign flow, removed-member badge, loading/empty/error states, a11y: display-name text, remove button aria-label)
