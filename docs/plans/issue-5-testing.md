# Issue #5 - Testing Report

## Tests Performed

| Test | Result | Notes |
|------|--------|-------|
| Valid form data → typed object | PASS | Returns correct shape and types |
| Missing field | PASS | Throws `Missing field: <key>` |
| File upload | PASS | Throws `Field <key> must be a string` |
| Empty string | PASS | Allowed (valid string) |
| Multiple missing fields | PASS | Reports first missing field |
| Unicode/special characters | PASS | Emoji, CJK, escapes handled |
| Very long strings (100k chars) | PASS | No truncation or overflow |
| Blob value (not File) | PASS | Rejected as non-string |
| Empty string vs missing field | PASS | Correctly distinguished |
| Whitespace-only strings | PASS | Treated as valid strings |
| All required fields present | PASS | Returns expected object |

## Integration Verification

| Route | Before | After |
|-------|--------|-------|
| `app/routes/issues/new.tsx` | `as unknown as` cast | `parseFormData` + optional field handling |
| `app/routes/assigned/new.tsx` | `as unknown as` cast | `parseFormData` + optional field handling |
| `app/routes/projects/id.new.tsx` | `as unknown as` cast | `parseFormData` + optional field handling |
| `app/routes/teams/projects/new.tsx` | `as` cast + duplicate import | `parseFormData`, duplicate removed |
| `app/routes/projects/new.tsx` | `as` cast + manual undefined checks | `parseFormData`, cleaner flow |

## Bugs Found & Fixed

- **Duplicate import**: `app/routes/teams/projects/new.tsx` had `TeamIcon` imported twice (lines 13 and 18). Removed the duplicate.
- **Silent NaN propagation**: `Number(body.priority)` on a non-numeric string would produce `NaN`. With `parseFormData`, at least we guarantee the input is a string; a future enhancement could add numeric validation.

## Verified Robust Against

- Missing required fields (throws immediately)
- File uploads masquerading as text fields (throws immediately)
- Empty form submissions (throws on first missing field)
- Unicode and special characters (handled correctly)
- Very large string inputs (no buffer issues)
- Blob values (rejected same as File)

## Remaining Risks

| Risk | Status | Mitigation |
|------|--------|------------|
| `Number(priority)` still accepts non-numeric strings | ACKNOWLEDGED | Returns `NaN`; consider adding `zod` or similar for schema validation in future |
| Optional fields could be `File` objects | MITIGATED | Optional fields use `typeof === "string"` check |
| FormData with duplicate keys | NOT A RISK | `FormData.get()` returns first value; duplicates are edge case in this UI |
