# STORY-9.3: Navigate from a My Tasks item to the card in its board (deep link)

**Epic:** EPIC-9

As an authenticated user, I want to click on a My Tasks item and be taken directly to that card within its board view, so that I can read the card's full detail and take action without manually finding the right board.

## Acceptance Criteria

**Given** I am viewing My Tasks and click on a task row **When** the navigation begins **Then** the browser navigates to the board view for that card's board, with the card detail panel open (URL form: `/boards/{boardId}/cards/{cardId}`).

**Given** I navigate to `/boards/{boardId}/cards/{cardId}` directly (e.g. from a copied link) and I am a member of that board **When** the page loads **Then** the board view renders and the card detail panel opens automatically; the URL remains stable and is shareable.

**Given** I navigate to `/boards/{boardId}/cards/{cardId}` and I am NOT a member of that board **When** the page loads **Then** I am shown an access-denied page ("You don't have access to this board") — the card's content is never revealed; the board's existence is not disclosed (same `404`-equivalent treatment as all board endpoints for non-members).

**Given** I navigate to `/boards/{boardId}/cards/{cardId}` where `cardId` does not belong to `boardId` **When** the page loads **Then** the board view renders normally, the card detail panel is not opened, and a dismissible "Card not found" toast/notification is shown to inform the user that the linked card could not be located on this board; the toast does not block board interaction and disappears on dismissal.

**Given** I navigate to `/boards/{boardId}/cards/{cardId}` where `boardId` does not exist **When** the page loads **Then** I see a not-found page (consistent with the board-not-found treatment elsewhere in the app).

**Given** I share the deep-link URL with another user who IS a member of the board **When** they navigate to it **Then** they see the board and card detail panel (their own authorization applies; the link itself carries no elevated privilege).

**Given** the deep link is followed and the card detail panel is loading **When** a loading state is in progress **Then** a loading indicator is shown within the panel; if the card load fails an error state is shown with a retry option.

**Given** I use keyboard navigation in My Tasks **When** I focus a task row and press Enter **Then** navigation to the deep link occurs (equivalent to clicking the row); the task row is keyboard-focusable and has an accessible name.

**Given** I interact with the deep-link task row, the board view, and the card detail panel **When** I use a keyboard or assistive technology **Then** all elements meet WCAG 2.2 AA: the task row and card detail panel are fully keyboard-navigable; all interactive elements have accessible names; focus moves to the panel heading when the panel opens and returns to the triggering element when it closes; color is not the sole visual differentiator for any state (open panel, active row, error state); `aria-label` or equivalent is present on the panel close button and all icon-only controls.

## NFR / constraints
- **Authorization on arrival:** the deep link DOES NOT bypass board authorization. The board view's existing authorization guard (EPIC-1, EPIC-3) applies on arrival; no new auth surface is introduced.
- **URL stability:** the route `/boards/{boardId}/cards/{cardId}` is a client-side route (Vue Router); the backend API serves the SPA on any sub-path. The `cardId` is resolved by an existing card-detail endpoint that already enforces board membership.
- **No new API endpoint:** this story is primarily a frontend routing and UX concern. It reuses existing endpoints: the board-load endpoint (EPIC-3) and the card-detail endpoint (established by earlier epics). The only new code is the Vue Router route definition and the deep-link open-on-mount behavior in the card detail panel component.
- **Accessibility (WCAG 2.2 AA):** each task row in the My Tasks list is a navigable element (`<a>` or `role="link"`) with an accessible name that includes the card title and board name (e.g. "Review onboarding copy — Board: Marketing"); screen-reader users can distinguish rows. The card detail panel: keyboard-navigable; focus moves to panel heading on open, returns to triggering row on close; accessible names on all controls (panel close, retry button, etc.); color is not the sole differentiator for any interactive or status state.
- **Clean Architecture:** no new Application or Infrastructure layer work; this story is constrained to the Api routing layer and the Vue frontend.

## UX spec (skeleton)
Each task row in the My Tasks list is rendered as a navigable link element pointing to `/boards/{boardId}/cards/{cardId}`. Clicking anywhere on the row (except explicit action buttons introduced in future stories) navigates to the board view. On arrival at the board view, if a `cardId` route param is present, the card detail panel opens automatically via an `onMounted` watch or route-param watcher. Panel open/close does not change the board URL base segment — only the `cards/{cardId}` suffix is added/removed (push/replace on Vue Router). Access-denied state: a full-page message ("You don't have access to this board") with a "Go to my boards" link. Not-found state: consistent with the app's existing 404 page. Loading state within the panel: spinner replacing the card detail content area. Error state: "Could not load card" with a Retry button. WCAG 2.2 AA: task rows are `<a>` elements (or equivalent with `role="link"` and `tabindex="0"`); accessible name set via `aria-label` combining card title + board name; focus management — when the panel opens focus moves to the panel heading; when the panel closes focus returns to the triggering element (if returning to My Tasks, to the task row).

## Out of scope
- Editing or updating the card from the deep-link panel (EPIC-4 owns card mutation).
- Sharing links with users outside the board — no public/guest link feature in v1 (§1.3 Out).
- Notification-driven deep links (email/push notifications are out of scope, §1.3 Out).
- Breadcrumb or "back to My Tasks" navigation affordance — useful but deferred (routing history handles it via the browser back button).

**Design:** inline UX spec above (skeleton — no external comp)
**Traces:** §1.4, §1.5, §1.6, §2.2
**Tests:** `MyTasksDeepLink.spec.ts` (Vue/Vitest — task row renders as a link with correct href `/boards/{boardId}/cards/{cardId}`; accessible name includes card title and board name; Enter key triggers navigation; board view opens card detail panel when `cardId` route param is present on mount; panel closes and param is removed on dismiss; cardId does not belong to boardId → board view renders normally AND a dismissible "Card not found" toast is shown; WCAG aa: task row keyboard-focusable, panel heading receives focus on open, focus returns to task row on panel close, color not sole differentiator for open-panel state); `MyTasksDeepLinkAccess.spec.ts` (Vue/Vitest — access-denied state renders when board returns 404/403; not-found state renders when boardId is unknown; card-not-found on board shows toast not crash); `BoardDeepLinkIntegrationTests` (Api integration — GET `/boards/{boardId}` for a non-member returns the same non-disclosure response regardless of whether a `cardId` is supplied in the path, confirming no authorization bypass)
