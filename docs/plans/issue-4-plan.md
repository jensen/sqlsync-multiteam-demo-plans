# Issue #4 – Issue Comments & Activity Feed

## Issue Summary

Add the ability to comment on issues and see an activity timeline of changes. This feature spans the Rust WASM reducer and the React frontend, consisting of 6 tasks with clear dependency ordering.

**Dependency Graph:**
- Tasks 1 & 2 (Rust reducer) → can run in parallel
- Tasks 3 & 4 (React UI) → depend on Tasks 1 & 2, can run in parallel
- Task 5 (Integration) → depends on Tasks 3 & 4
- Task 6 (Types) → depends on Tasks 1 & 2

## Root Cause Analysis

This is a **new feature request**, not a bug. The current system supports:
- Creating issues with title, body, status, priority, assignee
- Moving issues between projects
- Archiving/restoring issues

What is **missing**:
- No way to add threaded discussion/comments on issues
- No audit trail of who changed what and when
- No way to see the history of an issue's lifecycle

## Proposed Solution

### Task 1: Comment Schema & Mutations (`reducer/src/lib.rs`)

Add a `comments` table with foreign keys to `issues` and `users`:

```sql
CREATE TABLE IF NOT EXISTS comments (
    id TEXT PRIMARY KEY,
    issue_id TEXT NOT NULL,
    author_id TEXT NOT NULL,
    body TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (issue_id) REFERENCES issues(id),
    FOREIGN KEY (author_id) REFERENCES users(id)
)
```

Add mutations:
- `AddComment { id, issue_id, author_id, body }`
- `EditComment { id, body }`
- `DeleteComment { id }`

### Task 2: Activity Log Schema & Mutations (`reducer/src/lib.rs`)

Add an `activities` table:

```sql
CREATE TABLE IF NOT EXISTS activities (
    id TEXT PRIMARY KEY,
    issue_id TEXT NOT NULL,
    actor_id TEXT NOT NULL,
    action TEXT NOT NULL,
    payload TEXT,
    created_at TEXT NOT NULL,
    FOREIGN KEY (issue_id) REFERENCES issues(id)
)
```

Add mutation:
- `LogActivity { id, issue_id, actor_id, action, payload }`

**Hook into existing mutations** (`UpdateIssue`, `AssignIssue`, `MoveIssues`) to auto-insert activity rows. For example, when status changes from "backlog" → "inprogress", insert:
```
action: "status_changed", payload: '{"from":"backlog","to":"inprogress"}'
```

### Task 3: Comment UI Components (`app/routes/issues/components/comments/`)

Create:
- **`CommentList.tsx`** — scrollable list of comments for an issue, sorted by `created_at` ascending
- **`CommentInput.tsx`** — textarea + submit button for new comments
- **`CommentItem.tsx`** — single comment with:
  - Author avatar (using `UserIcon`)
  - Author name
  - Timestamp (relative, e.g. "2 hours ago")
  - Comment body
  - Edit/delete actions (visible only to author)

### Task 4: Activity Feed UI Component (`app/routes/issues/components/activity-feed.tsx`)

Create `ActivityFeed.tsx` that:
- Queries `activities` for the current issue
- Renders each activity as a timeline item with:
  - Actor avatar
  - Action description (e.g. "Alice changed status from Backlog to In Progress")
  - Timestamp
- Groups consecutive activities by the same actor

### Task 5: Integration (`app/routes/issues/components/issue.tsx`)

Update `issue.tsx` to:
- Add tabs or sections below the issue body: "Comments" | "Activity"
- Pass `issue_id` to comment components
- Query and render both comments and activities

### Task 6: TypeScript Types (`app/doctype.ts`)

Add to the `Mutation` union type:
- `AddComment`, `EditComment`, `DeleteComment`
- `LogActivity`

Add types:
```typescript
export type Comment = {
  id: string;
  issue_id: string;
  author_id: string;
  body: string;
  created_at: string;
  updated_at: string;
};

export type Activity = {
  id: string;
  issue_id: string;
  actor_id: string;
  action: string;
  payload: string | null;
  created_at: string;
};
```

## Files to Modify

| File | Change |
|------|--------|
| `reducer/src/lib.rs` | Add `comments` and `activities` tables, new mutations, auto-log activities |
| `app/doctype.ts` | Add `Comment`, `Activity` types and mutation variants |
| `app/routes/issues/components/issue.tsx` | Integrate comments and activity feed |
| `app/routes/issues/id.tsx` | Pass `issue_id` to child components |

## New Files

| File | Purpose |
|------|---------|
| `app/routes/issues/components/comments/CommentList.tsx` | List of comments |
| `app/routes/issues/components/comments/CommentInput.tsx` | Input for new comments |
| `app/routes/issues/components/comments/CommentItem.tsx` | Single comment display |
| `app/routes/issues/components/activity-feed.tsx` | Activity timeline |

## Data Flow

![Data Flow](./issue-4-data-flow.png)

## UI Mockup

![UI Mockup](./issue-4-mockup.png)

## Architecture

![Architecture](./issue-4-architecture.png)

## Test Strategy

1. **Reducer tests**: Verify `AddComment`, `EditComment`, `DeleteComment` mutations work correctly
2. **Reducer tests**: Verify activity auto-logging on status/assignee/project changes
3. **UI tests**: Verify comment CRUD operations
4. **UI tests**: Verify activity feed renders correctly
5. **Integration**: Verify comments sync across clients via SQLSync

## Risks

| Risk | Mitigation |
|------|-----------|
| Schema migration conflicts with existing data | `CREATE TABLE IF NOT EXISTS` is safe; new tables don't affect existing data |
| Activity logging performance overhead | Activities are simple inserts; SQLite handles this fine at demo scale |
| UI layout breaks with many comments | Use `overflow-y-auto` with max-height on comment list |
| Comment edit/delete permissions | Only show actions when `currentUser.id === comment.author_id` |
