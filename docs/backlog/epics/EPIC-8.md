# EPIC-8: Search & Filtering Within a Board

**Goal:** A board member can quickly narrow a board's cards by metadata attributes or freetext to find the work that matters, entirely within a single board, with server-enforced membership scoping, sub-1.5s performance on a ~200-card board, and a clear UX for active filters and empty results.
**FRs covered:** Search/filter (§1.3); Performance — board ~200 cards < ~1.5s, all list endpoints paginated (§1.6); Roles/membership enforced server-side (§1.4).
**Traces:** §1.3, §1.4, §1.6, §2.2

## Stories

- STORY-8.1 — Filter a board's cards by metadata (label, assignee, priority, due-date range)
- STORY-8.2 — Text-search a board's cards by title and description
- STORY-8.3 — Combine filters and text search, display active filter chips, and clear them

## Out of scope

- Cross-board search or aggregation (that is EPIC-9 — My Tasks).
- Saved/pinned filters, shareable filter URLs (future).
- Full-text indexing beyond case-insensitive LIKE (covered in STORY-8.2; upgrading to SQL FTS is noted as a future path).
- Searching comments, activity, or attachments.

## Notes

- All filter/search endpoints gate on board membership; non-members receive `404` (existence not disclosed) — consistent with §1.4 and the shared status convention.
- EPIC-4 (labels, assignees, due date, priority) is a hard prerequisite; these stories assume those fields exist.
- Performance budget for filter + search responses: < ~1.5s for a ~200-card board (§1.6). The index constraints in STORY-8.1 and STORY-8.2 satisfy this budget; combining them in STORY-8.3 reuses the same indexes.
