# Fronting Tracker MVP - Data Model Specification

## Purpose

This application allows a system to track which members are active ("fronting"), when they become active, and when they stop being active.

The MVP is designed for use by a single system on a single device with no user accounts, passwords, cloud synchronization, or network connectivity.

The data model should be designed to support future expansion without significant database refactoring, including:

* Avatars
* Member notes
* Roles
* Status messages
* Co-fronting
* Custom front types
* Session tags
* Custom fields
* Communication logs
* Polls and voting
* Change history
* App locking
* Cloud synchronization

---

# Terminology

## System

The collective group of members using the application.

There is exactly one system per installation in the MVP.

## Member

An individual system member.

## Ongoing Session

A period of time during which a member is considered active/fronting.

## Archived

A soft-delete state.

Archived records remain in the database but are hidden from normal operation.

---

# Global Rules

## Time Storage

All timestamps must be stored in UTC.

All timestamps must be displayed using the device's local time zone.

## Time Precision

Store and report times with minute precision.

## Soft Delete Strategy

Records should never be permanently deleted by normal application operations.

Instead:

* Active records have ArchivedAt = null
* Archived records have ArchivedAt populated

---

# Member Table

## Member

| Field      | Type                   | Notes       |
| ---------- | ---------------------- | ----------- |
| Id         | UUID                   | Primary key |
| Name       | String                 | Required    |
| Color      | String                 | Required    |
| isReal     | INT                    | Required    |
| CreatedAt  | UTC Timestamp          | Required    |
| UpdatedAt  | UTC Timestamp          | Required    |
| ArchivedAt | UTC Timestamp Nullable | Soft delete |

## Rules

### Name Uniqueness

Member names must be unique among active members.

Names may contain:

* Spaces
* Brackets
* Parentheses
* Special characters

Examples:

* Alex
* Alex [Work]
* Alex (Protector)

### Color

Colors do not need to be unique.

The application may warn users when duplicate colors are chosen.

### Member Archiving

Members cannot be permanently deleted through the normal UI.
Members can be 'Active' or 'Archived'

Archiving a member:
* Removes them from active member selection
* Preserves all historical data
* Preserves reporting for historical periods

---

# Session Table

## Session

| Field        | Type                   | Notes       |
| ------------ | ---------------------- | ----------- |
| session      | INT AUTOINCREMENT      | Primary key |
| MemberId     | INT AUTOINCREMENT      | Foreign key |
| StartTimeUtc | UTC Timestamp          | Required    |
| EndTimeUtc   | UTC Timestamp          | NULL = ongoing |
| CreatedAt    | UTC Timestamp          | Required    |
| UpdatedAt    | UTC Timestamp          | Required    |
| ArchivedAt   | UTC Timestamp Nullable | Soft delete |

---

# Session Rules

## Multiple Active Members

Multiple members may have active sessions simultaneously (both ongoing and completed).

## Same Member Overlap

Overlapping sessions for the same member are not allowed.

### Double Check-InO Sess
If a member already has an ongoing session and is checked in again:
1. Notify the user.
2. End the existing session at the new check-in time.
3. Start a new session beginning at the new check-in time.

## Check-Out Without Check-In

If a member is checked out while not in an ongoing session:
1. Notify the user.
2. Create a session with:
   * StartTime = checkout time
   * EndTime = checkout time
The user may manually edit the session later.

## Retroactive Session Creation
Users may manually create completed sessions.
Users may manually create ongoing sessions.

## Session Editing
Users may edit:
* Member
* Start time
* End time

Validation rules (below) must still apply.
A session cannot be changed from completed to ongoing or vice-versa without explicit confirmation.

## Session Validation
Reject sessions where:
* End time occurs before start time
* Session overlaps another session belonging to the same member
* There is no start time. Sessions must have a start time.

Display a clear explanation when validation fails.

## Session Archiving
Sesions can be 'Active' or 'Archived'
Archived sessions:
* Remain in the database
* Are hidden from normal views
* Are excluded from standard reports
* May be included in future specialized reports

---

# Ongoing Session Tracking
Sessions can be 'Ongoing' or 'Completed'
   - ongoing sessions have endTimeUTC = NULL
   - completed sessions have a non-null endTimeUTC

A member is considered to be in or have an ongoing session when:
* An ongoing session with their memberID exists 
   * StartTimeUtc is populated
   * EndTimeUtc has not yet been set

The application should support multiple active members simultaneously.

---

# Reporting Calculations

## Coverage Time

Coverage time is the amount of elapsed time during which at least one member had a session.
Coverage time is not the sum of member hours.

Example:
Alex: 09:00–10:00
Jamie: 09:30–10:30
Coverage Time = 1.5 hours
Not 2.0 hours.

---

## Unaccounted Time
Unaccounted time is any period in the reporting range during which no members had active sessions.

---

## Shared Time

Shared time is any period during which a member had a session simultaneously with at least one other member's session.

Example:
Alex: 09:00–12:00
Jamie: 10:00–11:00

Alex:
* Alone = 2 hours
* Shared = 1 hour

Jamie:
* Alone = 0 hours
* Shared = 1 hour

---

## Session Crossing Midnight

Sessions remain a single stored record.

For:
* Longest session
* Average session length
Use the original session duration.

For:
* Daily totals
* Weekly totals
* Monthly totals
* Coverage calculations

Split session time across the appropriate dates.

---

# Reporting Periods

The MVP supports:

* Today
* Yesterday
* DEFERRED Specific Day
* This Week
* This Month

---

# Future Tables

The following tables are expected in future versions and should be considered during architecture design:

## AuditLog

Tracks changes to members, sessions, and settings.

## MemberProfile

Stores avatars, notes, status messages, and future profile fields.

## SessionTag

Stores tags attached to sessions.

## Role

Stores member roles.

## FrontType

Stores custom fronting types.

## Poll

Stores polls and voting records.

## CommunicationLog

Stores internal communication records.

## CustomField

Stores user-defined profile and session fields.
