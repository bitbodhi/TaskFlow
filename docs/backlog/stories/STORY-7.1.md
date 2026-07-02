# STORY-7.1: Record activity events

**Epic:** EPIC-7

As the system, I want to append an immutable activity entry to a board's log whenever a member performs a mutation, so that the board's history is always accurate and in sync with reality.

## Acceptance Criteria

**Given** a board **Owner** or **Editor** successfully creates, updates, or deletes a **list** on the board **When** the mutation commits **Then** an activity entry is written in the same database transaction, recording: `boardId`, `actorId`, actor display name (snapshot), `eventType`, target entity id, a human-readable `description` snapshot (e.g. `"List 'Backlog' created"`), and `occurredAt` (UTC). The entry is visible in subsequent feed reads.

**Given** a board **Owner** or **Editor** successfully creates, updates (title, description, due date, priority, position/list), reorders within a list, or deletes a **card** **When** the mutation commits **Then** an activity entry is written in the same transaction with `eventType` from the card event types below and a snapshot `description` (e.g. `"Card 'Fix login bug' moved to 'In Progress'"` or `"Card 'Fix login bug' repositioned in 'In Progress'"`). The entry persists even if the card is later deleted.

**Given** a board **Owner** or **Editor** successfully reorders **lists** on the board **When** the mutation commits **Then** an activity entry is written in the same transaction with `eventType: list.reordered` and a snapshot `description` (e.g. `"Lists reordered"`). The entry is visible in subsequent feed reads.

**Given** a board **Owner** or **Editor** successfully adds or removes a **label** from a card, or creates/renames/deletes a **board-level label** **When** the mutation commits **Then** an activity entry is written with the appropriate `eventType` and description snapshot.

**Given** a board **Owner** or **Editor** successfully assigns or unassigns a **member** from a card **When** the mutation commits **Then** an activity entry is written recording the assignee's display name snapshot.

**Given** a board **Owner** or **Editor** successfully creates or deletes a **comment** on a card **When** the mutation commits **Then** an activity entry is written recording the comment's card title snapshot. (Comment edits are out of scope — see Out of scope.)

**Given** a board **Owner** adds or removes a **board member** or changes a member's role **When** the mutation commits **Then** an activity entry is written with the affected member's display name snapshot.

**Given** a board **Owner** renames the **board itself** **When** the mutation commits **Then** an activity entry is written with `eventType: board.renamed` and snapshots of the old and new name.

**Given** any mutation that would normally produce an activity entry **fails** (e.g. a validation error or a rollback) **Then** no activity entry is written (transactional atomicity guaranteed — the log cannot drift from reality).

**Given** a Viewer attempts any mutation endpoint **When** the request is processed **Then** the mutation is rejected (the mutation story's own auth enforces this) and therefore no activity entry is written — entry creation is strictly a side effect of successful commits.

## Event types recorded

| `eventType` | Trigger |
|---|---|
| `board.renamed` | Board title changed |
| `board.member_added` | Member joined the board |
| `board.member_removed` | Member removed from the board |
| `board.member_role_changed` | Member's role changed |
| `list.created` | List created |
| `list.renamed` | List title changed |
| `list.deleted` | List deleted |
| `list.reordered` | Lists reordered on the board |
| `card.created` | Card created |
| `card.renamed` | Card title changed |
| `card.moved` | Card moved to a different list |
| `card.repositioned` | Card reordered within the same list |
| `card.description_updated` | Card description set or changed |
| `card.due_date_set` | Card due date set |
| `card.due_date_removed` | Card due date cleared |
| `card.priority_changed` | Card priority changed |
| `card.deleted` | Card deleted |
| `card.label_added` | Label attached to a card |
| `card.label_removed` | Label detached from a card |
| `card.assignee_added` | Member assigned to a card |
| `card.assignee_removed` | Member unassigned from a card |
| `label.created` | Board-level label created |
| `label.renamed` | Board-level label renamed |
| `label.deleted` | Board-level label deleted |
| `comment.created` | Comment posted on a card |
| `comment.deleted` | Comment deleted |

## NFR / constraints
- **Atomicity:** the activity entry MUST be written in the same EF Core `SaveChanges` call (same unit of work / database transaction) as the mutation it records. The feed must never drift from the actual state of the board.
- **Immutability:** `ActivityEntry` rows have no update or delete paths in the application layer. No endpoint, background job, or migration may mutate existing rows. The EF entity has no setter for any field beyond the constructor.
- **Snapshot preservation:** actor display name and target label (e.g. card title, list name) are captured at write time so the entry remains meaningful if the source entity is later renamed or deleted.
- **Clean Architecture (§2.2):** `ActivityEntry` entity lives in Domain; `IActivityWriter` interface (or equivalent domain event / outbox pattern) lives in Application; the EF Core `ActivityEntries` table and write implementation live in Infrastructure. No EF or Microsoft.* namespace references in Domain.
- **No circular dependency:** mutation use-cases (EPIC-2 through EPIC-5) dispatch domain events or call `IActivityWriter`; the activity infrastructure handles persistence. Mutation handlers must not directly construct `ActivityEntry` rows with EF types.
- **Security / authorization:** the writer is called only from within authenticated, authorized use-cases — it trusts the `actorId` it is given and does not re-authorize.
- **RFC 7807 / DTOs:** no activity DTO is exposed by this story (read path is STORY-7.2). This story only concerns the write path.
- **Performance:** writing an entry must not add a perceptible round-trip; it is part of the same transaction as the mutation.
- **EF migration:** an `ActivityEntries` table is created by a new EF Core migration with columns: `Id` (PK, Guid), `BoardId` (FK), `ActorId` (FK → Users), `ActorDisplayName` (string, snapshot), `EventType` (string), `Description` (string), `OccurredAt` (DateTimeOffset UTC). Index on `(BoardId, OccurredAt DESC)` to support the read path efficiently.

## Out of scope
- Comment edits producing an activity entry (comment editing is out of v1.0 scope).
- System-generated or automated entries with no human actor.
- Activity for board creation or deletion (board creation is covered by EPIC-0/EPIC-1; board deletion is a separate story).
- The read (feed) endpoint — that is STORY-7.2.
- Real-time push of new entries to open clients — that is STORY-7.3.

**Traces:** §1.3, §1.5, §2.2
**Tests:** `ActivityEntryTests` (Domain unit — entity construction, immutability assertions, snapshot fields present); `RecordActivityHandlerTests` (Application unit — each event type produces a correctly shaped entry including `list.reordered` and `card.repositioned`, failed mutation produces no entry, actor snapshot captured); `ActivityWriterIntegrationTests` (Infrastructure integration — entry written in same transaction as mutation, rolled-back mutation produces no entry, `(BoardId, OccurredAt)` index present, `list.reordered` entry written when lists reordered, `card.repositioned` entry written when card reordered within same list); no UI in this story.
