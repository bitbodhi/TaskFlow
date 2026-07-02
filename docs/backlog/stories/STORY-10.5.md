# STORY-10.5: SPA reliability audit — loading/empty/error states + WCAG 2.2 AA sweep

**Epic:** EPIC-10

As an on-call engineer validating demo-readiness (and as an end-user), I want a time-boxed Definition-of-Done audit sweep across every SPA view — confirming that loading/empty/error states and WCAG 2.2 AA requirements (already mandated in each UI story's own ACs) are consistently implemented — so that the application is shippable and no view was silently skipped during EPIC-0 through EPIC-9 delivery.

> **Estimability note (fix 6 — NOT-INVEST resolved):** This story is a **time-boxed verification sweep**, not a new per-view implementation. Each UI story (EPIC-0 through EPIC-9) already owns its loading/empty/error + WCAG ACs; those stories are the implementation work. At sprint planning this story decomposes into a per-view checklist (one row per view × three states × WCAG gate); the time-box is **1 sprint spike (max 3 days)**. If a view is found non-compliant, a new bug story is filed — this story does NOT expand to fix it inline. This is a Definition-of-Done audit, not a re-implementation.

## Acceptance Criteria

**Given** any SPA view that fetches remote data (board list, board detail, card detail, "My Tasks", search results) **When** the fetch is in progress **Then** a loading indicator is visible (skeleton screen, spinner, or progress region); the indicator is announced to screen readers via `aria-busy="true"` or an `aria-live="polite"` region; the interactive controls that depend on the data are disabled or absent while loading.

**Given** any SPA view that fetches remote data **When** the API returns an empty collection or the resource does not exist **Then** a contextually appropriate empty-state message is shown ("No boards yet", "No cards in this list", etc.) with a primary call-to-action where applicable (e.g. "Create your first board"); the empty state is not a blank screen.

**Given** any SPA view that fetches remote data **When** the API returns a `4xx` or `5xx` response, or the network request fails **Then** an error state is shown with a human-readable message, a retry action, and — for errors that include an RFC 7807 `traceId` — the trace id surfaced as copy-able support reference; no raw JSON, stack trace, or internal error detail is displayed to the user.

**Given** an error state is displayed **When** a screen reader traverses the page **Then** the error message region has `role="alert"` or is an `aria-live="assertive"` region so the message is announced immediately on appearance.

**Given** any interactive element across all SPA views (form field, button, link, modal, drag-and-drop handle) **When** audited against WCAG 2.2 AA **Then** it passes: keyboard navigability (Tab/Shift-Tab/Enter/Space/Esc/arrow keys); visible focus indicator (CSS `:focus-visible` with ≥3:1 contrast on the focus ring); colour contrast ≥4.5:1 for normal text (≥3:1 for large text and UI components); no information conveyed by colour alone; form inputs have programmatically associated `<label>` elements; error messages are associated via `aria-describedby`.

**Given** a destructive action (delete board, delete card, remove member) **When** the user triggers it **Then** a confirmation dialog appears; the dialog is a modal with `role="dialog"`, `aria-modal="true"`, a visible title (`aria-labelledby`), and focus trapped inside; `Esc` cancels; confirming calls the API; the dialog closes after the action completes or errors.

**Given** the Sentry SPA integration (STORY-10.4) is active **When** an unhandled Vue component error surfaces the error state **Then** the component's `onErrorCaptured` hook (or the global `app.config.errorHandler`) captures the error to Sentry before rendering the error state — no silent swallowing.

**Given** an error state includes a `traceId` support reference **When** the user activates the copy-to-clipboard action **AND** the Clipboard API is unavailable (non-secure context, `navigator.clipboard` absent, or the permission is denied) **Then** the copy button is replaced by (or falls back to) a selectable, focusable `<input readonly>` or `<pre tabindex="0">` element containing the trace id so the user can manually select and copy it; the fallback element has a visible label ("Support reference — select to copy") and is keyboard-reachable.

## NFR / constraints

- This story is a **time-boxed Definition-of-Done audit sweep** (max 3 days / 1 spike). It is NOT a new feature and NOT a re-implementation. Findings of non-compliance result in new bug stories — they do not expand this story's scope.
- Sprint-planning decomposition: produce a per-view checklist (view × {loading, empty, error} × WCAG gate); each row is a pass/fail audit result. The checklist is the deliverable.
- **WCAG 2.2 AA** is the floor — specifically SC 1.4.3 (contrast), SC 1.4.11 (non-text contrast), SC 2.4.7 (focus visible), SC 2.4.11 (focus not obscured, AA in 2.2), SC 2.5.3 (label in name), SC 4.1.3 (status messages).
- Loading, empty, and error states must be implemented consistently using a **shared composition** (`useAsync` composable or Pinia store convention) rather than ad-hoc per-component flags, to prevent divergence as new views are added.
- Error messages shown to users must be derived from the RFC 7807 `title` field (if available) or a generic localised string — never the raw `detail` or exception message.
- The `traceId` support reference from RFC 7807 is surfaced as a copy-to-clipboard element, not a link, since it is not a URL.
- Automated accessibility testing: Lighthouse accessibility score ≥ 90 is the **CI hard gate** (per §2.7); this story must not regress below that threshold. Manual keyboard and screen-reader spot-checks supplement automation for dialog focus trap and `aria-live` announcements.
- **No external design comp** — the UX spec below is the design reference.

## UX spec (skeleton)

**Loading state:** a content skeleton (grey placeholder blocks matching the layout of the loaded content) with `aria-busy="true"` on the container. Duration: shown from request initiation until data is available or an error occurs.

**Empty state:** centred illustration area (SVG icon) + heading ("No boards yet") + body copy + primary action button ("Create board") where applicable. For views where creation is not in scope (e.g. "My Tasks" with no assignments), body copy only ("You have no tasks assigned across any board").

**Error state:** a card/banner with a warning icon, a one-sentence human message ("Something went wrong loading this board. Please try again."), a "Retry" button that re-triggers the fetch, and — when an RFC 7807 `traceId` is present — a collapsed "Support reference" disclosure (`<details>`/`<summary>`) containing the trace id with a copy button. The error card has `role="alert"`. Inline field errors use `aria-describedby`.

**Confirmation dialog:** modal overlay with `role="dialog"` and `aria-modal="true"`. Title (`aria-labelledby`): "Delete [item name]?" Body: consequence summary. Buttons: "Cancel" (left, secondary) / "Delete" (right, destructive/danger). Focus moves to the dialog on open (first interactive element); Tab cycles within; `Esc` = Cancel. After action: focus returns to the trigger element.

**Design:** inline UX spec above (skeleton — no external comp)

## Out of scope

- Adding new views or features; this story modifies existing views only.
- Internationalisation (i18n) of error strings — English only for v1.0.
- Offline / service-worker support.
- WCAG 2.2 AAA criteria.

**Traces:** §1.6, §1.2 ("loading/empty/error states"), §2.9 (traceId from RFC 7807 responses surfaced in UI)
**Tests:** `ErrorBoundary.spec.ts` + `LoadingState.spec.ts` + `EmptyState.spec.ts` (Vitest — each view's loading/empty/error branches rendered with mocked Pinia store states; assert `aria-busy`, `role="alert"`, `aria-live` present; assert error message does not contain raw exception text; assert traceId copy element renders when traceId present; `ClipboardFallback.spec.ts` — when `navigator.clipboard` is undefined or `writeText` rejects, the fallback selectable element is rendered, has a label, and is keyboard-focusable; when Clipboard API succeeds, the copy button is shown and the fallback element is absent); `ConfirmationDialog.spec.ts` (Vitest — focus trap; Esc closes; aria attributes present); Lighthouse CI gate ≥ 90 (existing gate, must not regress); audit checklist (manual) — per-view pass/fail rows documented as the spike deliverable.
