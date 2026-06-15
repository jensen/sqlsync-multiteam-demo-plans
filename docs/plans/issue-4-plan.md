# Issue #4 – Issue Comments & Activity Feed

## Issue Summary

Add the ability to comment on issues and see an activity timeline of changes. This feature spans the Rust WASM reducer and the React frontend.

## Current Status

The following tasks are **already completed** in the existing worktree (`fix/issue-4`):

| Task | Status | Location |
|------|--------|----------|
| Task 1: Comment Schema & Mutations | ✅ Complete | `reducer/src/lib.rs` |
| Task 2: Activity Log Schema & Mutations | ✅ Complete | `reducer/src/lib.rs` |
| Task 6: TypeScript Types | ✅ Complete | `app/doctype.ts` |
| Rust Integration Tests | ✅ Complete | `tests/dag-runs/*/integration_test.rs` |

**Remaining work:** Tasks 3, 4, and 5 — the React UI layer and integration.

## Root Cause Analysis

N/A — this is a feature request, not a bug.

## Architecture Overview

The feature uses SQLSync's local-first architecture: mutations flow from React → WASM reducer → SQLite, then sync to other clients.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  React UI   │────▶│  useMutate  │────▶│ WASM Reducer│
│  Components │     │  (sqlsync)  │     │   (lib.rs)  │
└─────────────┘     └─────────────┘     └──────┬──────┘
      ▲                                          │
      │                                          │
      │     ┌─────────────┐                    │
      └─────│  useQuery   │◀───────────────────┘
            │  (sqlsync)    │
            └─────────────┘
```

### Data Model

**Comments Table:**
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| issue_id | TEXT FK | References issues(id) |
| created_by | TEXT FK | References users(id) |
| body | TEXT | Comment content |
| created_at | TEXT | ISO timestamp |
| updated_at | TEXT | ISO timestamp (nullable) |

**Activities Table:**
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| issue_id | TEXT FK | References issues(id) |
| actor_id | TEXT FK | References users(id) |
| type | TEXT | Activity type enum |
| payload | TEXT | JSON metadata (nullable) |
| created_at | TEXT | ISO timestamp |

### Auto-Logged Activities

The reducer automatically inserts activity rows when certain mutations occur:

| Mutation | Activity Type | Payload |
|----------|--------------|---------|
| `AddIssue` | `created_issue` | null |
| `AssignIssue` | `assigned_issue` | `{ assigned_to, old_assigned_to, new_assigned_to }` |
| `UpdateIssue` | `updated_issue` | `{ changes: { status?, priority? } }` |
| `ArchiveIssues` | `archived_issue` | null |
| `RestoreIssues` | `restored_issue` | null |
| `MoveIssues` | `moved_issue` | `{ project_id, new_project_id }` |
| `AddComment` | `commented` | null |
| `UpdateComment` (trigger) | `updated_comment` | null |
| `DeleteComment` (trigger) | `deleted_comment` | null |

### Note on Naming

The issue specification uses `author_id` and `LogActivity`, but the existing implementation uses `created_by` and `AddActivity` to maintain consistency with the existing codebase patterns. This is a deliberate design choice — the functionality is identical.

## Proposed Solution

### Remaining Task 3: Comment UI Components

Create `app/routes/issues/components/comments/` directory with:

- **`CommentList.tsx`** — Fetches comments for the current issue via `useQuery`, renders a scrollable list of `CommentItem` components, sorted by `created_at` ascending.
- **`CommentInput.tsx`** — Textarea with submit button. Uses `useMutate` to dispatch `AddComment` mutations. Generates UUID via `crypto.randomUUID()` (or `uuid` package fallback).
- **`CommentItem.tsx`** — Displays author avatar (via `UserIcon`), name, timestamp, body. Includes edit/delete actions. Edit mode switches body to a textarea. Delete shows a confirmation state.

### Remaining Task 4: Activity Feed Component

Create `app/routes/issues/components/activity/` directory with:

- **`ActivityFeed.tsx`** — Fetches activities for the current issue via `useQuery`. Renders a chronological timeline with friendly text for each activity type:
  - `created_issue`: "Alice created this issue"
  - `assigned_issue`: "Alice assigned to Bob"
  - `updated_issue`: "Alice changed status from Backlog → In Progress"
  - `archived_issue`: "Alice deleted this issue"
  - `restored_issue`: "Alice restored this issue"
  - `moved_issue`: "Alice moved to Project X"
  - `commented`: "Alice commented"
  - `updated_comment`: "Alice edited a comment"
  - `deleted_comment`: "Alice deleted a comment"

Groups by date (Today, Yesterday, Earlier).

### Remaining Task 5: Integrate into Issue Detail Page

Update `app/routes/issues/components/issue.tsx`:
- Add tab/switcher: "Details" | "Comments (N)" | "Activity"
- Details tab shows current issue body + metadata
- Comments tab shows `CommentList` + `CommentInput`
- Activity tab shows `ActivityFeed`
- Pass `issue_id` down to comment/activity components

Update `app/routes/issues/id.tsx`:
- Query for `project_id` to pass through (already queried)

## Files to Modify

| File | Change |
|------|--------|
| `app/routes/issues/components/issue.tsx` | Add tabs, integrate comments and activity feed |
| `app/routes/issues/id.tsx` | Ensure `issue_id` is available (already is via params) |

## New Files

| File | Purpose |
|------|---------|
| `app/routes/issues/components/comments/CommentList.tsx` | Scrollable comment list |
| `app/routes/issues/components/comments/CommentInput.tsx` | New comment input |
| `app/routes/issues/components/comments/CommentItem.tsx` | Single comment display |
| `app/routes/issues/components/activity/ActivityFeed.tsx` | Activity timeline |

## Test Strategy

Since the project has no frontend test framework configured, we will:

1. **Verify Rust tests pass** — Run the existing integration tests in `tests/dag-runs/*/`
2. **Build verification** — Ensure `npm run build` succeeds
3. **Type checking** — Ensure `tsc --noEmit` (or equivalent) passes
4. **Manual UI verification** — Run `npm run dev` and verify:
   - Comments can be added, edited, deleted
   - Activity feed auto-populates on mutations
   - Tabs switch correctly
   - Dark theme styling is consistent

### Future Test Setup (out of scope)

Adding Vitest + React Testing Library would be ideal for component-level tests, but requires devDependencies not currently in the project.

## Risks

| Risk | Mitigation |
|------|------------|
| SQLSync schema mismatch between reducer and frontend | Ensure `InitSchema` is called before querying comments/activities |
| `useMutate` returns `null` in multi-document mode | Components must handle `mutate === null` gracefully (read-only) |
| Author name resolution | Query `users` table alongside comments to resolve `created_by` → name |
| Date formatting consistency | Use `Intl.DateTimeFormat` consistently with existing code |
| Activity payload parsing | Use `JSON.parse` with try/catch for safety |

## Acceptance Criteria Checklist

- [x] Users can add, edit, and delete comments on any issue (reducer ready, UI needed)
- [x] Comments appear in real-time across synced clients (SQLSync handles this)
- [x] Activity feed shows status changes, assignments, and moves automatically (reducer auto-logs)
- [ ] Comments and activities are sorted chronologically in the UI
- [ ] The UI matches the existing dark theme
- [x] All new code is TypeScript-typed correctly (types defined, UI to be typed)
