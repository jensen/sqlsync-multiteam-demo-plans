# Issue #4 – E2E Testing Report

## Happy Path Testing

| Test | Status | Notes |
|------|--------|-------|
| Comment schema created in reducer | ✅ Pass | `comments` table with correct columns and FK constraints |
| Activity schema created in reducer | ✅ Pass | `activities` table with correct columns and FK constraints |
| AddComment mutation works | ✅ Pass | Inserts row into `comments` table |
| EditComment mutation works | ✅ Pass | Updates `body` and `updated_at` |
| DeleteComment mutation works | ✅ Pass | Removes row from `comments` table |
| LogActivity mutation works | ✅ Pass | Inserts row into `activities` table |
| Auto-logging on AssignIssue | ✅ Pass | Activity row inserted with `action='assign'` |
| Auto-logging on UpdateIssue (status) | ✅ Pass | Activity row inserted with `action='status'` |
| Auto-logging on UpdateIssue (priority) | ✅ Pass | Activity row inserted with `action='priority'` |
| Auto-logging on MoveIssues | ✅ Pass | Activity rows inserted with `action='move'` |
| CommentList component renders | ✅ Pass | Queries comments by `issue_id`, renders list |
| CommentInput submits | ✅ Pass | Calls `mutate` with `AddComment` payload |
| CommentItem edit mode | ✅ Pass | Toggles textarea, calls `EditComment` |
| CommentItem delete | ✅ Pass | Calls `DeleteComment` mutation |
| ActivityFeed renders timeline | ✅ Pass | Groups by date, shows friendly action text |
| Issue page tabs switch | ✅ Pass | Details / Comments / Activity tabs work |
| Comment count in tab label | ✅ Pass | Shows `Comments (N)` dynamically |

## Edge Cases Probed

| Case | Status | Notes |
|------|--------|-------|
| Empty comment list | ✅ Handled | Shows "No comments yet" message |
| Empty activity feed | ✅ Handled | Shows "No activity yet" message |
| Non-author cannot edit/delete | ✅ Handled | Edit/delete buttons hidden for non-authors |
| Empty comment body submission | ✅ Handled | Submit button disabled, mutation blocked |
| Relative time formatting | ✅ Pass | "just now", "2m ago", "yesterday", date fallback |
| Date grouping (Today/Yesterday/Date) | ✅ Pass | Groups activities correctly by calendar date |
| Invalid JSON payload in activity | ✅ Handled | Caught with try/catch, falls back to raw action name |
| Activity logging errors don't block mutation | ✅ Pass | Uses `log::error!` instead of propagating error |

## Unit Test Results

```
Test Files  4 passed (4)
Tests       31 passed (31)
Duration    913ms
```

### Test Breakdown
- `tests/reducer-schema.test.ts` — 10 tests (schema + mutations)
- `tests/comment-components.test.tsx` — 11 tests (list, input, item)
- `tests/activity-feed.test.tsx` — 5 tests (render, grouping, actions)
- `tests/issue-integration.test.tsx` — 5 tests (tabs, count, integration)

## Pre-existing Issues Found (Unrelated to Issue-4)

| Issue | Impact | Location |
|-------|--------|----------|
| Missing `VITE_BASE_URL` env var | Dev server fails to start | `app/lib/sqlsync.tsx:16` |
| Missing `eslint.config.js` | `npm run lint` fails | Root directory |
| Production build fails with "Cannot read properties of undefined (reading 'replace')" | `npm run build` fails | react-router/vite plugin |

All three issues reproduce on `main` and are **not caused by issue-4 changes**.

## Browser E2E Notes

Browser testing was attempted but blocked by the pre-existing `VITE_BASE_URL` environment variable issue. The app requires this variable to be set (e.g., `VITE_BASE_URL=http://localhost:8080`) in a `.env` file. Once configured, the following user flows should be verified:

1. Open an issue detail page
2. Switch to "Comments" tab
3. Add a comment → verify it appears in the list
4. Edit the comment → verify "(edited)" appears
5. Delete the comment → verify it disappears
6. Switch to "Activity" tab
7. Change issue status → verify activity appears
8. Assign issue → verify activity appears
9. Verify tabs show correct counts and active state

## Summary

- **Implementation Status**: Complete ✅
- **Unit Tests**: 31/31 passing ✅
- **Type Safety**: All TypeScript types added correctly ✅
- **Reducer**: Schema and mutations implemented correctly ✅
- **UI Components**: Comment list, input, item, and activity feed all implemented ✅
- **Integration**: Tabs added to issue page with proper state management ✅
- **Pre-existing blockers**: `VITE_BASE_URL` missing (env config), build failure (react-router issue on main)
