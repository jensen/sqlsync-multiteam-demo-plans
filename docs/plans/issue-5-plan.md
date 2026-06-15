# Issue #5 – Fix: Type-erasing casts on formData() bypass runtime type safety

## Issue Summary

Five route action handlers parse `FormData` using `Object.fromEntries()` followed by type-erasing casts (`as unknown as { ... }` or `as { ... }`). `FormData` values are `string | File` at runtime — the casts provide zero validation and hide real type mismatches. If a `File` is uploaded or a field is missing, the code silently propagates bad data into mutations.

## Affected Files

| File | Line | Current Pattern |
|------|------|-----------------|
| `app/routes/issues/new.tsx` | 10 | `as unknown as { ... }` |
| `app/routes/assigned/new.tsx` | 11 | `as unknown as { ... }` |
| `app/routes/projects/id.new.tsx` | 10 | `as unknown as { ... }` |
| `app/routes/teams/projects/new.tsx` | 21 | `as { ... }` (no `unknown`) |
| `app/routes/projects/new.tsx` | 17 | `as { ... }` (partial validation) |

## Root Cause Analysis

1. **False type safety**: The `as` casts tell TypeScript the data has a specific shape, but at runtime `FormData` entries can be:
   - `File` objects instead of strings
   - Missing fields (resulting in `undefined`)
   - Malformed values from tampered requests

2. **Inconsistent patterns**: Some files use `as unknown as`, others use bare `as`, showing the codebase lacks a shared convention.

3. **Silent failures**: When `body.name` is a `File` instead of a string, `name.trim()` may throw or behave unexpectedly. When `priority` is not a valid number string, `Number(body.priority)` returns `NaN` which gets stored silently.

## Proposed Solution

### 1. Create a shared `parseFormData` helper

Create `app/lib/form.ts` with a generic helper that:
- Accepts a `FormData` and an array of required field names
- Validates each field is present and is a `string` (not `File`)
- Returns a typed object with all specified fields guaranteed to be strings
- Throws descriptive errors for missing or invalid fields

```typescript
export function parseFormData<T extends Record<string, string>>(
  formData: FormData,
  keys: (keyof T)[]
): T {
  const result = {} as T;
  for (const key of keys) {
    const value = formData.get(key as string);
    if (value === null) throw new Error(`Missing field: ${String(key)}`);
    if (typeof value !== "string") throw new Error(`Field ${String(key)} must be a string`);
    result[key] = value as T[keyof T];
  }
  return result;
}
```

### 2. Replace all unsafe casts

Replace each `Object.fromEntries(await request.formData()) as ...` with a call to `parseFormData`.

### 3. Fix duplicate import bug

`app/routes/teams/projects/new.tsx` has a duplicate `TeamIcon` import (lines 13 and 18). Remove the duplicate.

### 4. Add runtime validation for optional fields

For optional fields like `project_id` and `assigned_to`, validate them separately after the required fields are parsed.

## Files to Modify

| File | Change |
|------|--------|
| `app/lib/form.ts` | **New file** — `parseFormData` helper |
| `app/routes/issues/new.tsx` | Replace unsafe cast with `parseFormData` |
| `app/routes/assigned/new.tsx` | Replace unsafe cast with `parseFormData` |
| `app/routes/projects/id.new.tsx` | Replace unsafe cast with `parseFormData` |
| `app/routes/teams/projects/new.tsx` | Replace unsafe cast + remove duplicate import |
| `app/routes/projects/new.tsx` | Replace unsafe cast with `parseFormData` |

## Test Strategy

1. **Unit tests for `parseFormData`**: Install Vitest (standard for Vite projects) and create `tests/form.test.ts` with tests for:
   - Valid form data → returns typed object
   - Missing field → throws error
   - File upload → throws error
   - Empty string → allowed (valid string)
   - All required fields present with correct types

2. **Integration tests**: Verify each modified action still works with valid form data.

## Risks

| Risk | Mitigation |
|------|------------|
| Helper throws on missing optional fields | Parse optional fields separately after required fields |
| `Number(priority)` could still receive non-numeric strings | Keep `Number()` conversion but `parseFormData` ensures it's at least a string |
| Tests fail due to no existing test framework | Install Vitest as dev dependency |

## Diagram

![Before/After](./issue-5-before-after.png)
