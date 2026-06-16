# Issue #3 – Add Issue Labels (Tags) with Team Management, Picker, and Filtering

## Issue Summary

Add **labels (tags)** to issues: a team can define a set of colored labels, apply multiple labels to any issue, see them as chips, and filter the issue list by label. A core issue-tracker capability (à la Linear/GitHub) that's currently missing.

**Motivation:** Teams need a lightweight way to categorize issues (`bug`, `feature`, `urgent`, `frontend`, …) beyond status/priority, to speed up triage and filtering.

## Root Cause Analysis

This is a **feature request**, not a bug. The current system lacks:
- No label data model (tables for `label` and `issue_label`)
- No mutations for label CRUD operations
- No UI components for label management, picking, or display
- No filtering capability by labels

## Proposed Solution

### Data Model

Add two new tables to the schema:

```sql
-- Label definition (scoped to team document)
CREATE TABLE label (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    color TEXT NOT NULL,
    created_at TEXT NOT NULL
);

-- Many-to-many relationship between issues and labels
CREATE TABLE issue_label (
    issue_id TEXT NOT NULL,
    label_id TEXT NOT NULL,
    PRIMARY KEY (issue_id, label_id),
    FOREIGN KEY (issue_id) REFERENCES issues(id),
    FOREIGN KEY (label_id) REFERENCES label(id)
);
```

### Mutations (Rust + TypeScript)

**Rust (`reducer/src/lib.rs`):**
```rust
enum Mutation {
    // ... existing mutations ...
    
    // Label management
    AddLabel { id: String, name: String, color: String },
    RenameLabel { id: String, name: String, color: String },
    RemoveLabel { id: String },  // Also deletes issue_label rows
    
    // Label assignment
    AddLabelToIssue { issue_id: String, label_id: String },
    RemoveLabelFromIssue { issue_id: String, label_id: String },
}
```

**TypeScript (`app/doctype.ts`):**
```typescript
type Mutation =
  // ... existing mutations ...
  | { tag: "AddLabel"; id: string; name: string; color: string }
  | { tag: "RenameLabel"; id: string; name: string; color: string }
  | { tag: "RemoveLabel"; id: string }
  | { tag: "AddLabelToIssue"; issue_id: string; label_id: string }
  | { tag: "RemoveLabelFromIssue"; issue_id: string; label_id: string };

type Label = {
  id: string;
  name: string;
  color: string;
  created_at: string;
};
```

### UI Components

1. **Label Management Screen** (`app/routes/teams/[teamid]/labels.tsx`)
   - List all labels for the team
   - Create new label (name + color picker)
   - Edit existing label (rename, recolor)
   - Delete label (with confirmation)

2. **Label Picker Component** (`app/routes/issues/components/label-picker.tsx`)
   - Multi-select dropdown
   - Shows available labels with colored chips
   - Used in issue create and issue detail views

3. **Label Chips Display**
   - Inline component showing assigned labels as colored pills
   - Appears on issue list rows and issue detail view

4. **Label Filter** (`app/routes/issues/components/label-filter.tsx`)
   - Filter issue list by one or more selected labels
   - Multi-select with label chips

## Files to Modify

### Backend (Rust WASM Reducer)
- `reducer/src/lib.rs` - Add label mutations and schema

### TypeScript Types
- `app/doctype.ts` - Add Label type and mutation variants

### New UI Components
- `app/routes/issues/components/label-picker.tsx` - Multi-select for labels
- `app/routes/issues/components/label-chip.tsx` - Colored label display
- `app/routes/issues/components/label-filter.tsx` - Filter by labels
- `app/routes/teams/[teamid]/labels.tsx` - Label management screen

### Modified UI Components
- `app/routes/issues/components/create.tsx` - Add label picker to create form
- `app/routes/issues/components/details.tsx` - Add label management to issue details
- `app/routes/issues/components/list.tsx` - Show label chips on rows, add filter
- `app/routes/issues/id.tsx` - Load labels for issue detail

## Test Strategy

### Unit Tests
1. **Reducer tests** (if test infrastructure exists):
   - Label CRUD operations
   - Add/remove label from issue
   - Cascade delete (removing label removes issue_label rows)

2. **Component tests**:
   - Label picker multi-select behavior
   - Label chip rendering with correct colors
   - Filter applies correct SQL WHERE clauses

### Integration Tests
1. Create label → appears in picker
2. Apply label to issue → shows in chips
3. Filter by label → correct issues shown
4. Delete label → removed from all issues
5. Sync across tabs (two clients see same label changes)

### Manual Testing Checklist
- [ ] Create/rename/recolor/delete labels in management screen
- [ ] Add/remove multiple labels on issue create
- [ ] Add/remove labels from issue detail view
- [ ] Label chips show on issue detail and list rows
- [ ] Filter issue list by one or more labels
- [ ] Deleting a label removes it from all issues
- [ ] Labels sync across clients (two tabs)
- [ ] Labels are isolated per team

## Risks

1. **Schema migration**: The system uses `create table if not exists` - adding tables is safe, but existing documents won't have label data until the InitSchema runs again. This is acceptable for new features.

2. **Cascade deletes**: SQLite doesn't enforce FK cascade by default. The `RemoveLabel` mutation must explicitly delete from `issue_label` table.

3. **Performance**: Filtering by multiple labels requires JOINs. For large datasets, may need indexing on `issue_label(issue_id, label_id)`.

4. **Color accessibility**: Users may pick low-contrast colors. Consider enforcing minimum contrast ratios or providing preset accessible color palettes.

5. **Sync conflicts**: Two users editing the same label simultaneously could cause conflicts. SQLSync's CRDT approach should handle this, but edge cases may exist.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         RENDERER PROCESS                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Label Mgmt   │  │ Label Picker │  │ Label Filter │          │
│  │   Screen     │  │  Component   │  │  Component   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └─────────────────┴─────────────────┘                   │
│                           │                                     │
│                    useMutate() / useQuery()                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │ IPC
┌───────────────────────────┼─────────────────────────────────────┐
│                    WASM REDUCER         │                       │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────┐           │
│  │              Mutation Handler                     │           │
│  │  ┌─────────────┐  ┌─────────────┐                │           │
│  │  │  AddLabel   │  │ RemoveLabel │                │           │
│  │  │ RenameLabel │  │ AddLabelTo  │                │           │
│  │  │             │  │ RemoveLabel │                │           │
│  │  └─────────────┘  └─────────────┘                │           │
│  └────────────────────────┬─────────────────────────┘           │
│                           │                                     │
│                    execute!(SQL)                                │
└───────────────────────────┼─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                      SQLITE DATABASE                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   issues    │    │    label    │    │ issue_label │         │
│  │             │    │             │    │             │         │
│  │ - id        │    │ - id        │    │ - issue_id  │         │
│  │ - title     │    │ - name      │    │ - label_id  │         │
│  │ - status    │    │ - color     │    │             │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Sequence

1. **Phase 1: Schema + Mutations** (Backend)
   - Add `label` and `issue_label` tables to InitSchema
   - Implement label CRUD mutations in Rust
   - Add TypeScript types in doctype.ts

2. **Phase 2: Label Management UI** (Team Settings)
   - Create label management screen
   - Label list with edit/delete
   - Create new label modal with color picker

3. **Phase 3: Label Picker + Chips** (Components)
   - Create reusable LabelChip component
   - Create LabelPicker multi-select component

4. **Phase 4: Integration** (Issue Create/Detail)
   - Add label picker to issue create form
   - Add label management to issue detail view
   - Show label chips on issue detail

5. **Phase 5: List + Filter** (Issue List)
   - Show label chips on issue list rows
   - Add label filter component
   - Implement SQL filtering by labels

6. **Phase 6: Testing + Polish**
   - Write unit/integration tests
   - Test sync across clients
   - Accessibility review (color contrast)
   - Performance optimization (indexes if needed)

## Acceptance Criteria

- [ ] Create / rename / recolor / delete a label
- [ ] Add/remove multiple labels on an issue (at create time and from the detail view)
- [ ] Label chips show on the issue detail and on each issue-list row
- [ ] The issue list can be filtered by one or more labels
- [ ] Deleting a label removes it from all issues (`issue_label` rows cleaned up)
- [ ] Labels and assignments sync across clients (two tabs; changes propagate)
- [ ] Labels are isolated per team

## Out of Scope

- Label-based automation / saved views
- Cross-team / global labels
- Bulk label editing from list multi-select
