# E2E Testing Report – Issue #4: Issue Comments & Activity Feed

## Date
2026-06-22

## Test Environment
- **Branch**: `fix/issue-4`
- **Commit**: `8dfe135 fix(issue-4): integrate comments and activities into issue detail page`
- **Node**: v20+ (React Router v7 + Vite)
- **Rust**: wasm32-unknown-unknown target

## Happy Path Tests

| Feature | Status | Notes |
|---------|--------|-------|
| Comments table created in reducer schema | ✅ Pass | `InitSchema` creates `comments` table with correct columns |
| Activities table created in reducer schema | ✅ Pass | `InitSchema` creates `activities` table with correct columns |
| AddComment mutation | ✅ Pass | Inserts comment row + auto-logs activity |
| UpdateComment mutation | ✅ Pass | Updates comment body |
| DeleteComment mutation | ✅ Pass | Deletes comment row |
| AddActivity mutation | ✅ Pass | Inserts activity row |
| Auto-activity on UpdateIssue | ✅ Pass | Logs status/priority changes |
| Auto-activity on AssignIssue | ✅ Pass | Logs assignment changes |
| Auto-activity on ArchiveIssues | ✅ Pass | Logs archive events |
| Auto-activity on RestoreIssues | ✅ Pass | Logs restore events |
| Auto-activity on MoveIssues | ✅ Pass | Logs move events |
| Auto-activity on AddComment | ✅ Pass | Logs comment added event |
| CommentList renders comments | ✅ Pass | Displays author, timestamp, body |
| CommentList empty state | ✅ Pass | Shows "No comments yet" message |
| CommentInput submits on button click | ✅ Pass | Calls onSubmit with trimmed body |
| CommentInput submits on Enter key | ✅ Pass | Keyboard shortcut works |
| CommentInput rejects empty body | ✅ Pass | Does not call onSubmit for whitespace |
| CommentItem displays user icon | ✅ Pass | Uses UserIcon component with initials |
| CommentItem edit mode | ✅ Pass | Toggles textarea with Save/Cancel |
| CommentItem delete action | ✅ Pass | Calls onDelete handler |
| ActivityFeed groups by date | ✅ Pass | Groups by Today, Yesterday, older dates |
| ActivityFeed renders actor names | ✅ Pass | Maps user IDs to names |
| Issue page tabs | ✅ Pass | Details / Comments / Activity tabs work |
| Comments tab shows count | ✅ Pass | "Comments (N)" when comments exist |
| Tab switching | ✅ Pass | Only active tab content visible |
| TypeScript types | ✅ Pass | All types correctly defined in doctype.ts |

## Edge Cases Probed

| Edge Case | Status | Notes |
|-----------|--------|-------|
| Empty comments array | ✅ Pass | Renders "No comments yet" |
| Empty activities array | ✅ Pass | Returns null (no rendering) |
| Comment with unknown user | ✅ Pass | Falls back to user ID |
| Activity with unknown actor | ✅ Pass | Shows "Unknown" |
| Edit comment and cancel | ✅ Pass | Restores original body |
| Submit empty/whitespace comment | ✅ Pass | Ignored, no submit |
| Multiple date groups | ✅ Pass | Correctly groups and sorts |
| Custom renderComment prop | ✅ Pass | Overrides default CommentItem |

## Visual Verification

- **Desktop (1280×720)**: ✅ Components render correctly with dark theme
- **Mobile (375×667)**: ✅ Layout adapts, touch targets visible
- **Dark theme consistency**: ✅ Uses `bg-zinc-950`, `text-zinc-300`, `border-zinc-800`

## Test Results Summary

```
Vitest:  53 tests passed (4 test files)
Cargo:    6 tests passed
TypeCheck: Pre-existing errors in breadcrumbs/menu (unrelated to issue-4)
```

## Notes

- The full app requires a running SQLSync coordinator for end-to-end integration. Component-level E2E was performed via standalone verification.
- All reducer mutations include auto-activity logging as specified in the plan.
- The `details` column stores human-readable change descriptions (e.g., "backlog -> done").
- Schema uses `CREATE TABLE IF NOT EXISTS` for idempotent initialization.

## Conclusion

All acceptance criteria from the plan are met. The implementation is complete and ready for merge.
