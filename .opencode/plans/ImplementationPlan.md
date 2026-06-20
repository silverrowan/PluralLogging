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
  - Seed a default "Unknown" member (grey color) on first launch.
  - Unknown is a real DB member, hidden from the normal member list, always available in check-in.
- **`db/memberQueries.ts`**:
  - `getAll()` — non-archived members
  - `getAllActive()` — non-archived members who are NOT currently fronting
  - `insert(name, color)` — returns new member
  - `update(id, name, color)`
  - `archive(id)` — sets archivedAt
- **`db/sessionQueries.ts`**:
  - `getActiveMembers()` — sessions with endTimeUtc=NULL, JOIN members, exclude archived
  - `startSession(memberId, startTime)` — insert session with NULL endTime
  - `endSession(sessionId, endTime)` — set endTimeUtc
  - `getHistory(startDate, endDate)` — past sessions in range
  - `updateSession(sessionId, fields)` — edit member/start/end
  - `endAllActiveSessions(endTime)` — end all currently active sessions

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
    - Prominent "Check In" button
    - "Add All" / "Remove All" buttons side by side

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
- **Multi-select**: Long-press an ActiveMemberCard to enter selection mode → tap multiple cards → floating "Check Out Selected" button → ConfirmDialog

### Step 7 — Member Management
- **`app/members/index.tsx`**:
  - List of all non-archived members (color, name)
  - FAB to add member
  - Tap to edit, swipe or long-press to archive
- **`MemberFormDialog.tsx`**:
  - TextInput for name (required, 1-200 chars)
  - 30+ preset color swatches in a scrollable grid
  - Toggle: "Show Unknown in member list" (controls visibility of the Unknown member)
  - Save button (disabled until name filled)

### Step 8 — Session History
- **`app/history/index.tsx`**:
  - List of past sessions grouped by date
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
- Check-out without check-in: create zero-length session (per spec)
- UTC storage, local time display
- Handle database errors with Paper Snackbar

## Key Design Decisions

| Decision | MVP Choice |
|----------|-----------|
| Unknown member | Real DB member, seeded on first launch. Grey (#9E9E9E), named "Unknown". Available in check-in. Optionally visible in member list. |
| Color picker | 30+ preset swatches in a scrollable grid |
| Check-out confirmation | Individual = instant. Remove All / multi-select = confirmation dialog. |
| Undo | Deferred (not MVP) |
| Multi-select checkout | Include: long-press to enter selection mode, bulk checkout with confirmation |
| Session editing | Tap past session → edit member/start/end with validation |
| First launch | Empty state with CTA to add first member |
| Time precision | Minute precision, UTC storage, local display |

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS members (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  color TEXT NOT NULL,
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT
);

CREATE TABLE IF NOT EXISTS sessions (
  id TEXT PRIMARY KEY,
  memberId TEXT NOT NULL,
  startTimeUtc TEXT NOT NULL,
  endTimeUtc TEXT,
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL,
  archivedAt TEXT,
  FOREIGN KEY (memberId) REFERENCES members(id)
);

CREATE INDEX IF NOT EXISTS idx_sessions_memberId ON sessions(memberId);
CREATE INDEX IF NOT EXISTS idx_sessions_active ON sessions(endTimeUtc)
  WHERE endTimeUtc IS NULL;
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
