# Fronting Tracker MVP — Implementation Plan

## Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | Expo SDK 52+ (TypeScript) |
| Navigation | expo-router (stack navigator, no tabs) |
| Database | expo-sqlite 3.8.0+ or later |
| UI | React Native Paper (Material Design 3) |
| Icons | @expo/vector-icons (MaterialCommunityIcons) |

## File Structure

```
FrontingTracker/
├── app/
│   ├── _layout.tsx           # Root layout (Stack nav, DB provider)
│   ├── index.tsx             # HOME - Ongoing Sessions dashboard
│   ├── members/
│   │   ├── index.tsx         # Member list (add/edit/archive)
│   │   └── [id].tsx          # Member detail (MVP: simple view)
│   ├── history/
│   │   └── index.tsx         # Session history & editing
│   └── reports/
│       └── index.tsx         # Reports (today/week/month)
├── components/
│   ├── OngoingSessionCard.tsx  # Colored card: name, color dot, ✕ checkout
│   ├── CheckInSheet.tsx      # Bottom sheet: multi-select non-fronting members
│   ├── MemberFormDialog.tsx  # Add/edit member (name, color picker)
│   ├── EmptyState.tsx        # Reusable "nothing here" placeholder
│   └── ConfirmDialog.tsx     # Reusable "are you sure?" dialog
├── db/
│   ├── database.ts           # Init DB, create tables, seed Unknown member
│   ├── memberQueries.ts      # Member CRUD
│   └── sessionQueries.ts     # Session CRUD, ongoing session queries
├── hooks/
│   ├── useDatabase.ts        # SQLite DB context provider
│   ├── useOngoingSessions.ts # Currently fronting sessions and members
│   └── useMembers.ts         # All non-archived members
├── types/
│   └── index.ts              # Member, Session, MemberSession types
└── utils/
    └── time.ts               # UTC helpers, formatting
```

## Build Order

### Step 1 — Project Init
```
npx create-expo-app@latest FrontingTracker --template blank-typescript
```
Install dependencies:
- expo-router, expo-sqlite
- react-native-paper, react-native-safe-area-context, react-native-screens
- @expo/vector-icons
- expo-status-bar

### Step 2 — Database Layer
- **`db/database.ts`**: Open DB, run CREATE TABLE for `members` and `sessions`.
  - Seed Unknown member (isReal = 0) on first launch; special case: for unknown/disputed fronting, unlike other members permits multiple concurrent ongoing sessions (warning shown).
- **`db/memberQueries.ts`**:
  - `getMembers()` — non-archived members; excludes Unknown member (isReal = 0);
  - `insert(name, color)` — inserts new member and returns new member row, returns new member
  - `update(id, name, color)`  — updates member details
  - `archive(id)` — sets archivedAt timestamp for soft delete
- **`db/sessionQueries.ts`**:
  - `getOngoingSessions()` - returns all ongoing sessions (endTimeUTC == NULL)
  - `getCheckInList()` — returns non-archived members who are NOT currently fronting plus "unknown member" (isReal = 0)
    - getMembers() then REMOVE all members with memberIDs IN getOngoingSessions()
    - always add in unknown member (isReal = 0), even if present in getOngoingSessions() 
  - `startSession(memberId, startTime)` — inserts session with NULL endTimeUtc, for ongoing/fronting state
  - `endSession(sessionId, endTime)` — set endTimeUtc to timestamp
  - `getHistory(startDate, endDate)` — past sessions in range  WHERE endTimeUtc IS NOT NULL
  - `updateHistorySession(sessionId, fields)` — edit member/start/end; validates only if endTime set and passes checkHistoryEdit().
    - note: cannot change completed sessions to ongoing without explicit consent. 
  - `updateOngoingSession(sessionId, fields)` — edit member/start; Cannot end ongoing session. Validates only if endTime NULL, and passes checkOngoingEdit().
    - note: cannot end member(s)' ongoing sessions without explicit consent. 
  - `checkHistoryEdit()` — returns boolean; validates a completed session edit. 
    - If isReal = 0 then return true.
    - If new start and end times do not overlap existing sessions with the same memberID as this session (including both completed and ongoing) then return true.
      - Note: sessions may overlap provided they have different memberIDs.
    - otherwise return false.
  - `checkOngoingEdit()` — returns boolean; validates an ongoing session edit. 
    - If isReal = 0 then return true.
    - If new start time (to current time) does not overlap existing sessions with the same memberID as this session then return true.
      - Note: sessions may overlap provided they have different memberIDs.
    - otherwise return false.
  - `endAllOngoingSessions(endTime)`  — set endTimeUtc to timestamp for all ongoing sessions (including all unknown member sessions) else do nothing. -- Called by Remove All UI confirmation dialog.

### Step 3 — Types + Context
- **`types/index.ts`**: `Member`, `Session`, `MemberSession` (Session + Member joined)
- **`hooks/useDatabase.ts`**: React Context providing the DB instance
- Wrap `_layout.tsx` with the provider
- **`app/_layout.tsx`**: Root screen navigator structure:
  - Stack navigator from expo-router:
    - Root stack with substacks: Home, Members, History, Reports
    - Header navigation between all tabs; bottom nav bar optional (deferred)
  - Error boundary / error handler for crashes: show EmptyState or custom error modal
  - Global providers: SQLite DB context, theme provider

### Step 4 — Home Screen (Ongoing Dashboard)
- **`loading behaviour`**:
  - Screen shows "fetching fronting members" EmptyState while initial getOngoingSessions() query runs (~100ms).
  - Individual session cards are shown when query completes, not before.
  - Bottom Check In button shows Loading Spinner until member fetch completes.
- **`app/index.tsx`**:
  - Header buttons: [Members] [History] [Reports]
  - Title: "Currently Fronting"
  - Empty state: "No one is fronting" when no ongoing sessions
  - List of `OngoingSessionCard` components:
    - Color swatch, member name, ✕ checkout button
    - Show "Unknown" as separate OngoingSessionCard per each overlapping ongoing session (multiple cards with same gray dot/name, start times listed individually)
      - Each Unknown card's ✕ button ends only that specific session; multiple ongoing Unknown sessions appear one at a time until all checked out via individual or Remove All actions
  - Bottom area:
    - Prominent "Check In" button (enables multiple members to front simultaneously)
    - "Remove All" button only (no "Add All"; add members individually via Check-In sheet) -- note an 'Add All' button is DEFERRED

### Step 5 — Check-In Flow
- **`CheckInSheet.tsx`** (Paper Modal):
  - Title: "Check In"
  - Lists all non-fronting, non-archived members - includes 'unknown' (isReal = 0)
  - Each row: color dot + name + checkbox
  - "Select All" toggle
  - "Confirm" button — A session is created for each member selected, with startTimeUTC = now
    - note: members already fronting are not in the list and cannot be selected.
  - Cancel to dismiss

### Step 6 — Check-Out Flows  
- **Individual**: Tap ✕ on OngoingSessionCard → instant endSession (no confirmation)
- **Remove All**: Tap "Remove All" → ConfirmDialog → If confirmed end all ongoing sessions
- **Multi-select checkout**: DEFERRED to future. MVP supports only individual check-outs; multiple members can still be fronting simultaneously.

### Step 7 — Member Management
- **`app/members/index.tsx`**:
  - List of all non-archived members (color, name)
  - FAB to add member
  - Tap to edit; archive toggle in MemberFormDialog (not swipe/long-press gestures)
- **`MemberFormDialog.tsx`**:
  - TextInput for name (required, 1-200 chars)
  - 30+ preset color swatches in a scrollable list
  - Save button (disabled until name filled)

### Step 8 — Session History
- **`loading behaviour`**:
  - state "fetching recent sessions" displays during query (~10-50ms).
- **`app/history/index.tsx`**:
  - List of past sessions grouped by date
    - Default: sort order by: start time, then end time. Most recent first.
    - Default: limit to sessions with end times within the last 72 hrs
  - Each row: member name, color, start-end times, duration
  - Tap session to open edit view:
    - Change member, start time, end time
    - Validation: end > start, no overlapping sessions for same member
  - Search/filter by member name
    - empty search results shows: "no fronting sessions matching these filters are recorded"
  - Add new session button

### Step 9 — Reports
- **`loading behaviour`**:
  - state "fetching sessions" displays during query.
  - Period selector change triggers new aggregation query (~50ms).
  - Stats cards show skeleton blocks until results computed.
- **`app/reports/index.tsx`**:
  - Period selector: Today | Yesterday | This Week | This Month
  - a day is defined as midnight to midnight in the local user's time. 
    - that means today will not be calculated from a full 24hrs
  - Stats cards:
    - Coverage time (total time minus time where no sessions are active)
    - Unaccounted time (total time no sessions are active)
    - Total session hours
  - Optional: breakdown by member
    - Total session hours
    - Total session hours shared with other member(s)
    - Longest session
    - Average session length
  - Empty state "no fronting sessions matching these filters are recorded" for zero-match queries.
  - Note for report calculations and display:
    - sessions that cross midnight (day change) are stored as a single record, but are handled differently for reports.
      - Longest session and Average session length use the full length of the sessions for calculations, and are included if their endTimeUTC is within the reporting period
      - all totals and coverage calculations cut off session times that fall outside their range.  Sessions are included if EITHER OF (or both) startTimeUTC and endTimeUTC are within the reporting period. (IF startTimeUTC is before rangeStartUTC use rangeStartUTC. IF endTimeUTC is after rangeEndUTC use rangeEndUTC)

### Step 10 — Polish & Edge Cases
- First launch: if no members exist (beyond Unknown), show EmptyState with "Add your first member" CTA
- Double check-in prevention: only "unknown" member and non-fronting members shown in check-in sheet
- Check-out without check-in: create session record with both start/end time set to checkout timestamp (zero-duration entry); shouldn't be possible in UI.
- check-in list filtering handles check-in when already checked-in problem.
- UTC storage, local time display
- Single-device usage eliminates race conditions, transaction isolation; no network error handling needed
- Database errors captured and shown via Paper Snackbar.
- No catch-all async/await error swallowing; user-visible snackbar on SQL failures (constraint violations, foreign keys).
- report tab shows empty state "no recorded fronting sessions match these filters" when zero open matches are found per selected period + other filters
- *Note: Data export deferred to Post-MVP phase (see Future Features). Database layer already export-ready.

## Key Design Decisions

| Decision | MVP Choice |
|----------|------------|
| Color picker | 30+ preset swatches in a scrollable list |
| Time precision | Minute precision, UTC storage (ISO-8601 UTC (e.g., "2025-06-19T14:30:00Z")), local display; (enforcing of time format deferred) |
| Data export | DEFERRED — highest priority Post-MVP |
| Undo | DEFERRED (not MVP) |
| First launch | Empty state with CTA to add first member |
| Session Display | Scrollable list of wide and short cards. 1 card per session with member name, color, and an "x" button to end the session; session cards for unknown member (isReal = 0) additionally show session start time. Default sort by start time. | 
| Session Overlap | Sessions may overlap each other, provided they have different memberIDs, or isReal = 0 ('unknown' member) |
| Check-out Confirmation | On group actions only |
| Remove-all Action | First confirms intent to end all ongoing sessions with user. If confirmed then sets endTime to current time for all ongoing sessions, including all unknown member (isReal = 0) sessions, else does nothing |
| Add All or Multiple Action | DEFERRED Multi-select check-in, and add all check-in deferred. |
| Session editing | In session history: Tap past session → edit member/start/end with validation |
| Offline Design | See section "Offline-First Design" |

## Offline-First Design
-SQLite database is purely local; all queries execute on-device with full transaction support. 
- No server connection required; app functions 100% without network.
- Background sync / push notifications not applicable to MVP (local-only).
- File Access API-based backup available Post-MVP for cross-device migration.
- EVENTUAL option to add cloud or other multi-device. DEFERRED

## Data Integrity
- Foreign key constraints enforce referential integrity (sessions -> members).
- CHECK constraint prevents invalid endTimeUtc values.
- Soft-delete via archivedAt column preserves historical data without space loss.

## Future Features (High priority, considered in design; NOT IMPLEMENTED) 

### Data Export & Backup [PRIORITY: HIGHEST]
- SQLite offline backup/restore using File Access API  
- JSON export for cross-device sync foundation
- **Rationale:** MVP database is already export-ready. Implementation deferred now to focus on core tracking flows, but architecture supports full export capability without refactoring.

### (Post-MVP — High Priority Next Phases)
- add check to datetime columns to enforce correct time format.
- change tap to edit member to tap to view member details. edit button on viewpage.
- add Member Setting or Toggle: 'show in check-in' IN ADDITION TO active/archive status
- add option to hide 'unknown' fronter from ui
- additional confirmation, history, report, and edit rules to handle 'unknown' member 
- add option to set "priority" fronters, hide other fronters behind a "more members" option in member list.
- Home screen widget(s) with current fronters, Quick check-in/check-out
- User customization: ability to add fields and/or tags for users and/or sessions, eg. 'roles' 'mood' 'state' etc.
- Allow user to specify specific days and periods for history & reporting
- Allow user to change day roll-over time from default midnight(LocalTime) - (for those with different sleep schedules)
- Audit log
- expanded member profile, including images
- implement archive old sessions functionality

## Database Schema
```sql
CREATE TABLE IF NOT EXISTS members (
  memberId INT AUTOINCREMENT PRIMARY KEY,  -- hidden from user
  name TEXT NOT NULL,
  color TEXT NOT NULL,  -- users pick from 30+ swatches for each member
  isReal INT NOT NULL DEFAULT 1 CHECK(isReal IN (0, 1)), -- 0 = Unknown member or other recording entity that is not a member; currently only "unknown" member uses isReal = 0, and no other entities that do so are planned for MVP
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT -- NULL = active
);

Seed: INSERT INTO members (name, color, isReal, createdAt, updatedAt) VALUES ('Unknown', '#9E9E9E', 0, datetime('now'), datetime('now'))
-- "Unknown" user: seeded once on launch, never auto-archived; row information: isReal = 0; name = "Unknown"; color =  "#9E9E9E" ; createdAt = updatedAt = time at launch; archivedAt = NULL

CREATE TABLE IF NOT EXISTS sessions (
  sessionId INT AUTOINCREMENT PRIMARY KEY, -- hidden from UI and user
  memberId INT NOT NULL REFERENCES members(memberId),
  startTimeUtc TEXT NOT NULL,
  endTimeUtc TEXT, -- NULL = ongoing/fronting; ISO timestamp = completed session
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT, -- NULL = not Archived

  FOREIGN KEY (memberId) REFERENCES members(memberId),
  CHECK(endTimeUtc IS NULL OR endTimeUtc >= startTimeUtc)  -- ensure valid duration
);

CREATE INDEX IF NOT EXISTS idx_sessions_memberId ON sessions(memberId);
-- note ongoing sessions intentionally not indexed, assume < 50 ongoing sessions at any one point
CREATE INDEX IF NOT EXISTS idx_sessions_ended ON sessions(endTimeUtc) 
  WHERE endTimeUtc IS NOT NULL;
```