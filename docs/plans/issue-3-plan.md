# Issue #3 — Add Issue Labels (Tags)

## Issue Summary

Add **labels (tags)** to issues: a team can define a set of colored labels, apply multiple labels to any issue, see them as chips, and filter the issue list by label. This is a core issue-tracker capability (à la Linear/GitHub) that is currently missing.

## Scope / Requirements

- **Define labels per team** — create, rename, recolor, delete in a management screen
- **Apply labels to issues** — assign/unassign multiple labels on issue create and detail view
- **See labels** — chips on issue detail and on each issue-list row
- **Filter** — narrow the issue list by one or more selected labels
- Labels are scoped to a team (they live in that team's SQLSync document)

## Root Cause Analysis

This is a **feature request**, not a bug. The current data model supports users, projects, and issues, but has no concept of labels. The mutation enum in the reducer and the TypeScript types both need extension.

## Proposed Solution

### Data Model

Add two new tables in the `InitSchema` arm of the reducer:

```sql
create table if not exists label (
    id text primary key,
    name text not null,
    color text not null,
    created_at text not null
)

create table if not exists issue_label (
    issue_id text not null,
    label_id text not null,
    primary key (issue_id, label_id),
    foreign key (issue_id) references issues(id),
    foreign key (label_id) references label(id)
)
```

### Mutations (Rust + TypeScript)

| Mutation | Payload | Behavior |
|----------|---------|----------|
| `AddLabel` | `{ id, name, color }` | Insert into `label` |
| `UpdateLabel` | `{ id, name, color }` | Update `label` row |
| `RemoveLabel` | `{ id }` | Delete from `label`; also delete matching `issue_label` rows (no FK cascade) |
| `AddLabelToIssue` | `{ issue_id, label_id }` | Insert into `issue_label` |
| `RemoveLabelFromIssue` | `{ issue_id, label_id }` | Delete from `issue_label` |

### UI Components

1. **LabelChip** (`app/components/label/chip.tsx`) — colored pill showing label name
2. **LabelPicker** (`app/components/label/picker.tsx`) — multi-select dropdown for choosing labels
3. **LabelManager** (`app/routes/teams/labels.tsx`) — screen to CRUD team labels
4. **LabelFilter** — integrated into the issue list filter bar

### Files to Modify

| File | Change |
|------|--------|
| `reducer/src/lib.rs` | Add `label`/`issue_label` tables to `InitSchema`; add 5 new mutation arms |
| `app/doctype.ts` | Add `Label` type; extend `Mutation` union with 5 new variants |
| `app/routes/issues/components/create.tsx` | Add `LabelPicker` to create form; pass selected labels to action |
| `app/routes/issues/components/details.tsx` | Add `LabelPicker` for changing labels |
| `app/routes/issues/components/issue.tsx` | Display `LabelChip`s for the issue |
| `app/routes/issues/components/list.tsx` | Show `LabelChip`s on each row; add label filter |
| `app/routes/teams/issues.tsx` | Update SQL to join `issue_label`/`label`; support label filter param |
| `app/routes/issues/new.tsx` | Handle `label_ids` in form data; call `AddLabelToIssue` mutations |
| `app/routes.ts` | Add `teams/:teamid/labels` route |

### New Files

| File | Purpose |
|------|---------|
| `app/components/label/chip.tsx` | Reusable colored label pill |
| `app/components/label/picker.tsx` | Multi-select label dropdown |
| `app/routes/teams/labels.tsx` | Label management screen |

## Test Strategy

The project currently has **no test infrastructure** (no test runner, no test scripts in `package.json`). Given the scope, we will verify through:

1. **Type checking** — `tsc --noEmit` to ensure TypeScript correctness
2. **Rust compilation** — `cargo build` to ensure reducer compiles
3. **Manual browser verification** — run the dev server and exercise the UI
4. **Adversarial testing** — rapid label CRUD, edge cases (empty name, same color, rapid toggles)

## Risks

| Risk | Mitigation |
|------|----------|
| Rust/TS type mismatch | Keep enums in sync; use `serde(tag)` pattern consistently |
| No FK cascade on delete | Explicitly delete `issue_label` rows in `RemoveLabel` mutation |
| Label color contrast | Use a curated set of 12 preset colors; no free-form hex input |
| Race conditions on rapid toggle | Mutations are serialized by SQLSync coordinator |
| Filter state persistence | Use URL query params for filter state (already used for `filter` param) |

## Acceptance Criteria

- [ ] Create / rename / recolor / delete a label
- [ ] Add/remove multiple labels on an issue (at create time and from detail view)
- [ ] Label chips show on the issue detail and on each issue-list row
- [ ] The issue list can be filtered by one or more labels
- [ ] Deleting a label removes it from all issues (`issue_label` rows cleaned up)
- [ ] Labels and assignments sync across clients (two tabs; changes propagate)
- [ ] Labels are isolated per team

## Diagrams

![Data Model](./issue-3-data-model.png)

![UI Flow](./issue-3-ui-flow.png)
