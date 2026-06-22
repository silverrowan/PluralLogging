# Future Wants (Post-MVP)

> Deferred features and enhancements discussed during MVP planning.

## High Priority

### Home Screen Widget
- Android homescreen widget showing currently fronting members
- Quick check-in/check-out from widget
- Deferred from MVP, mentioned as eventual desire

### Undo for Check-Out
- Undo option after individual check-out, Remove All, and multi-select
- Could be a snackbar with "Undo" button that appears for ~5 seconds after action
- Deferred for simplicity in MVP

### Confirmation on All Check-Outs
- Long-term: confirmation dialogs on Remove All and multi-select check-out
- MVP uses instant for individuals, confirmation for bulk only

### Predefined Member Groups
- User-created named member sets (e.g., "Work team", "Social crew")
- Add/Remove group feature alongside individual and Add All
- May link with roles (user choice)
- Deferred from MVP, which only has individual + select-all

## Medium Priority

### iOS Support
- Cross-platform deployment to iOS
- Currently Android-first (Pixel 8), desktop debugging via web
- Eventually: Windows, macOS too

### Reports Expansion
- More report periods (custom date range, yearly)
- Visual charts (pie/bar for member breakdown)
- Export to CSV/PDF
- Archived session inclusion in specialized reports

### Session Tags
- Tags attached to sessions (e.g., "switching", "stressful", "quiet")
- Future table: `SessionTag`

## Future Tables (from MVPGuidedDescription.md)

### AuditLog
- Track changes to members, sessions, and settings
- Change history for accountability

### MemberProfile
- Avatars (images or initials)
- Member notes / descriptions
- Status messages
- Future profile fields

### Role
- Member roles (e.g., Protector, Caretaker, Gatekeeper)
- Role-based grouping for add/remove

### FrontType
- Custom fronting types
- User-defined categories for fronting states

### Poll
- Internal polls and voting records
- Group decision-making

### CommunicationLog
- Internal communication records between members

### CustomField
- User-defined profile and session fields
- Extensible data model

## Lower Priority / Polish

### Better Color Picker
- MVP: 30+ preset swatches
- Future: HSL color wheel/slider for unlimited color choice

### Fronting States / Switching Association
- "Add All" / "Remove All" linked to fronting states like "switching"
- Quick state-based batch operations

### App Locking
- PIN/biometric lock for the app
- Privacy/security for sensitive fronting data

### Notification for Double Check-In
- Spec says "Notify the user" when member already active is checked in again
- Toast/snackbar notification
- MVP avoids this by only showing non-fronting members in check-in sheet

### Cloud Synchronization
- Cross-device sync for the same system
- Requires user accounts, network connectivity
- Significant architectural change from local-only MVP
