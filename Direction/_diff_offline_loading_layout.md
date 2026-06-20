# Proposed Changes: Address File Structure Gaps

## Gap Summary & Proposed Insertions

---

### GAP 1: Missing app/_layout.tsx Implementation Details (Section 3 - Step 3)

**Current State:** Line 78-79 only mentions wrapping _layout.tsx with provider
```
- **`hooks/useDatabase.ts`**: React Context providing the DB instance  
- Wrap `_layout.tsx` with the provider
```

**Proposed Addition** (after line 79, before `### Step 4 — Home Screen`):
```markdown
- **`app/_layout.tsx`**: Root screen navigator structure:
  - Stack navigator from expo-router:
    - Root stack named (tabs) with substacks: Home, Members, History, Reports
    - Header navigation between all tabs; bottom nav bar optional (deferred)
  - Error boundary / error handler for crashes: show EmptyState or custom error modal
  - Global providers: SQLite DB context, theme provider
  - Root render: null on empty root state while loading

**Loading State Note:**
- All screens use native react-native Paper LoadingIndicator + skeleton patterns pending queries; SQLite read operations can take 50-300ms depending on data volume.
- Query examples causing load: getActiveMembers() (joined session lookup), getHistory() with filter, reports aggregation query.
- Skeleton states required for list items until hydrated from DB.

---


---

### GAP 3: Missing Offline-First Considerations Beyond Line 153/146

**Current State:** Line 146 and line 158 mention offline briefly but lack specific guidance:
```markdown
(Separate notes at):
- "Single-device usage eliminates race conditions, transaction isolation; no network error handling needed" (line 146)
- "Data export deferred to Post-MVP phase" (line 150)
```

**Proposed Addition** (in Section 9 / Step 9 or as separate subsection after Step 10):

Create new `### Step 10a — Offline-First Notes`:

```markdown
Offline-First Design:
- SQLite database is purely local; all queries execute on-device with full transaction support.
- No server connection required; app functions 100% without network.
- Background sync / push notifications not applicable to MVP (local-only).
- File Access API-based backup available Post-MVP for cross-device migration.

Error Handling Strategy:
- Database errors captured and shown via Paper Snackbar (configurable show/hide toggle per Step 146).
- No catch-all async/await error swallowing; user-visible snackbar on SQL failures (constraint violations, foreign keys).
- Query timeout not expected (< 300ms worst case on Pixel 8 SQLite); no timeout handling needed.

Data Integrity:
- Foreign key constraints enforce referential integrity (sessions -> members).
- CHECK constraint prevents invalid endTimeUtc values.
- Soft-delete via archivedAt column preserves historical data without space loss.
```

---

## Summary of New Sections

Insertions required in three places:

1. **After Step 3** (~line 80): `app/_layout.tsx` + Loading state note
2. **After Steps 4, 8, 9**: Inline loading behavior callouts  
3. **New subsection after Step 10** (~line 146): Expanded offline-first notes

Total additional lines: ~80-100 lines (markdown comments)

Would you like me to prepare the exact before/after code blocks, or proceed with applying these changes?