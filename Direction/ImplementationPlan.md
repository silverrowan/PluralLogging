# Fronting Tracker MVP — Implementation Plan

## Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | Expo SDK 52+ (TypeScript) |
| Navigation | expo-router (stack navigator, no tabs) |
| Database | expo-sqlite |
| UI | React Native Paper (Material Design 3) |
| Icons | @expo/vector-icons (MaterialCommunityIcons) |

## File Structure

```
FrontingTracker/
├── app/
│   ├── _layout.tsx           # Root layout (Stack nav, DB provider)
│   ├── index.tsx             # HOME - Active members dashboard
│   ├── members/
│   │   ├── index.tsx         # Member list (add/edit/archive)
│   │   └── [id].tsx          # Member detail (MVP: simple view)
│   ├── history/
│   │   └── index.tsx         # Session history & editing
│   └── reports/
│       └── index.tsx         # Reports (today/week/month)
├── components/
│   ├── ActiveMemberCard.tsx  # Colored card: name, color dot, ✕ checkout
│   ├── CheckInSheet.tsx      # Bottom sheet: multi-select non-fronting members
│   ├── MemberFormDialog.tsx  # Add/edit member (name, color picker)
│   ├── EmptyState.tsx        # Reusable "nothing here" placeholder
│   └── ConfirmDialog.tsx     # Reusable "are you sure?" dialog
├── db/
│   ├── database.ts           # Init DB, create tables, seed Unknown member
│   ├── memberQueries.ts      # Member CRUD
│   └── sessionQueries.ts     # Session CRUD, active session queries
├── hooks/
│   ├── useDatabase.ts        # SQLite DB context provider
│   ├── useActiveMembers.ts   # Currently fronting members
│   └── useMembers.ts         # All non-archived members
├── types/
│   └── index.ts              # Member, Session, ActiveMember types
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
  - Seed a default "Unknown" member (name as the string "unknown" grey color #9E9E9E) on first launch.
  - Secret member ID: treated as any other member, hidden from UI until user adds their own members.
- **`db/memberQueries.ts`**:
  - `getAll()` — all non-archived members (includes secret Unknown with proper ID handling)
  - `insert(name, color)` — inserts new member and returns new member row including unique ID for Unknown type
  - `update(id, name, color)` — updates member details
  - `archive(id)` — sets archivedAt timestamp for soft delete
- **`db/sessionQueries.ts`**:
  - `getActiveMembers()` — returns non-archived members who are NOT currently fronting (joined table filtered by NULL endTimeUtc)
  - `startSession(memberId, startTime)` — inserts session with NULL endTimeUtc for active/fronting state
  - `endSession(sessionId, endTime)` — set endTimeUtc to timestamp (only for completed sessions)
  - `getHistory(startDate, endDate)` — past sessions in range WHERE endTimeUtc IS NOT NULL
  - `updateSession(sessionId, fields)` — edit member/start/end; validates only if endTime set
  - `endAllActiveSessions(endTime)` — set endTimeUtc to timestamp for all active sessions

### Step 3 — Types + Context
- **`types/index.ts`**: `Member`, `Session`, `ActiveMember` (Session + Member joined)
- **`hooks/useDatabase.ts`**: React Context providing the DB instance
- Wrap `_layout.tsx` with the provider

### Step 4 — Home Screen (Active Dashboard)
- **`app/index.tsx`**:
  - Header buttons: [Members] [History] [Reports]
  - Title: "Currently Fronting"
  - Empty state: "No one is fronting" when no active sessions
  - List of `ActiveMemberCard` components:
    - Color swatch, member name, ✕ checkout button
  - Bottom area:
    - Prominent "Check In" button (enables multiple members to front simultaneously)
    - "Remove All" button only (no "Add All"; add members individually via Check-In sheet)

### Step 5 — Check-In Flow
- **`CheckInSheet.tsx`** (Paper Modal):
  - Title: "Check In"
  - Lists all non-fronting, non-archived members (Unknown included)
  - Each row: color dot + name + checkbox
  - "Select All" toggle
  - "Confirm" button — creates sessions for all selected members with startTimeUtc = now
  - Cancel to dismiss

### Step 6 — Check-Out Flows  
- **Individual**: Tap ✕ on ActiveMemberCard → instant endSession (no confirmation)
- **Remove All**: Tap "Remove All" → ConfirmDialog → end all active sessions
- **Multi-select checkout**: Deferred to future. MVP supports only individual check-outs; multiple members can still be fronting simultaneously.

### Step 7 — Member Management
- **`app/members/index.tsx`**:
  - List of all non-archived members (color, name)
  - FAB to add member
  - Tap to edit; archive toggle in MemberFormDialog (not swipe/long-press gestures)
- **`MemberFormDialog.tsx`**:
  - TextInput for name (required, 1-200 chars)
  - 30+ preset color swatches in a scrollable grid
  - Save button (disabled until name filled)

### Step 8 — Session History
- **`app/history/index.tsx`**:
  - List of past sessions grouped by date
    - Default: sort order by: start time, then end time. Most recent first.
    - Default: limit to sessions with end times within the last 72 hrs
  - Each row: member name, color, start-end times, duration
  - Tap to open edit view:
    - Change member, start time, end time
    - Validation: end > start, no overlapping sessions for same member
  - Search/filter by member name

### Step 9 — Reports
- **`app/reports/index.tsx`**:
  - Period selector: Today | This Week | This Month
  - Stats cards:
    - Coverage time (total time with at least one member active)
    - Unaccounted time (gaps with no activity)
    - Total session hours
  - Optional: breakdown by member

### Step 10 — Polish & Edge Cases
- First launch: if no members exist (beyond Unknown), show EmptyState with "Add your first member" CTA
- Double check-in prevention: only non-fronting members shown in check-in sheet
- Check-out without check-in: create session record with both start/end time set to checkout timestamp (zero-duration entry)
- UTC storage, local time display
- Handle database errors with Paper Snackbar

*Note: Data export deferred to Post-MVP phase (see Future Features). Database layer already export-ready.*

## Future Features (Post-MVP — Highest Priority Next Phase)

- add datetime checks to relevant columns.
- change tap to edit to tap to view. edit button on viewpage.
- Setting or Toggle: "Show Unknown in member list" (controls visibility of the Unknown member)
- add 'show in check-in' IN ADDITION TO active/archive status

### Data Export & Backup [PRIORITY: HIGHEST]
- SQLite offline backup/restore using File Access API  
- JSON export for cross-device sync foundation
- **Rationale:** MVP database is already export-ready. Implementation deferred now to focus on core tracking flows, but architecture supports full export capability without refactoring.




## Key Design Decisions

| Decision | MVP Choice |
|----------|------------|
| Unknown member | Real DB member with unique UID, seeded on launch. Grey (#9E9E9E), always available in check-in. Hidden from UI until user adds own members. |
| Color picker | 30+ preset swatches in a scrollable grid |
| Check-out confirmation on group actions only | Individual = instant. Remove All = confirmation dialog. Multi-select checkout deferred (multi-member fronting supported). |
| Undo | Deferred (not MVP) |
| Multi-select checkout | Deferred (not MVP) |
| Session editing | Tap past session → edit member/start/end with validation |
| First launch | Empty state with CTA to add first member |
| Time precision | Minute precision, UTC storage, local display |
Data export | Deferred — highest priority Post-MVP |

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS members (
  memberId INT AUTOINCREMENT PRIMARY KEY,  
  name TEXT NOT NULL,  
  color TEXT NOT NULL,  -- Unknown is grey (#9E9E9E); users pick from 30+ swatches for other members

  -- Store as ISO-8601 UTC (e.g., "2025-06-19T14:30:00Z")
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT
);

CREATE TABLE IF NOT EXISTS sessions (
  sessionId INT AUTOINCREMENT PRIMARY KEY, -- hidden from user, used by database to ensure unique sessions
  memberId TEXT NOT NULL REFERENCES members(id),  -- secret "uid_unknown" handled in DB layer

  -- Store as ISO-8601 UTC (e.g., "2025-06-19T14:30:00Z")
  startTimeUtc TEXT NOT NULL,
  endTimeUtc TEXT, -- NULL = active/fronting; ISO timestamp = completed session
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT,
  FOREIGN KEY (memberId) REFERENCES members(id),
  CHECK(endTimeUtc IS NULL OR endTimeUtc > startTime)  -- ensure valid duration
);

CREATE INDEX IF NOT EXISTS idx_sessions_memberId ON sessions(memberId);
CREATE INDEX IF NOT EXISTS idx_sessions_active 
    ON sessions(endTimeUtc) WHERE endTimeUtc IS NULL; 
```

## Color Swatches (30+)

```
#F44336  #E91E63  #9C27B0  #673AB7
#3F51B5  #2196F3  #03A9F4  #00BCD4
#009688  #4CAF50  #8BC34A  #CDDC39
#FFEB3B  #FFC107  #FF9800  #FF5722
#795548  #9E9E9E  #607D8B  #E57373
#F06292  #CE93D8  #9575CD  #7986CB
#64B5F6  #4DD0E1  #4DB6AC  #81C784
#AED581  #DCE775  #FFF176  #FFD54F
#FFB74D  #FF8A65  #A1887F  #90A4AE
```
