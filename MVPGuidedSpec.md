
1. DATA MODELS (with types, relationships, constraints)
2. UI FLOWS (user stories with acceptance criteria)
3. API CONTRACTS (endpoints, request/response shapes)
4. STATE MANAGEMENT (where data lives, how it updates)
5. EDGE CASES (error states, loading, empty states)
6. TECH STACK (exact versions, libraries, patterns)
7. NON-NEGOTIABLES (security, accessibility, performance)

## Data Models

### Description of what to put here
Don't just list fields. Define types, validation rules, relationships, and state transitions.
Why this works: AI now knows exact validation rules, foreign keys, and deletion behavior. It won't guess.

### Examples
#### Bad
Users have tasks w due dates

#### Good
# User Model
User:
  id: string (UUID, auto-generated)
  email: string (unique, validated format, max 255 chars)
  created_at: timestamp (ISO 8601, UTC)
  
# Task Model  
Task:
  id: string (UUID)
  user_id: string (foreign key to User.id)
  title: string (required, min 1, max 200 chars)
  description: string (optional, max 2000 chars)
  status: enum ['pending', 'in_progress', 'completed', 'archived']
  due_date: date (optional, must be >= today)
  created_at: timestamp
  updated_at: timestamp (auto-update on any change)
  
# Relationships
- User has many Tasks (1:N)
- Task belongs to one User
- Deleting a User soft-deletes their Tasks (set status='archived')

## UI Flows w Acceptance Criteria
### Description of what to put here
Use Given-When-Then format for each user story. 
This makes AI generate testable code.
Why this works: AI knows exact UI behavior, validation timing, success/failure UX, and ordering rules.

### Examples
#### Bad
User can add a task

#### Good
Story 1: Add a new task
  Given I am logged in and on the dashboard
  When I click "Add Task"
  Then I see a modal with fields: Title (required), Description, Due Date
  And the "Save" button is disabled until Title is filled
  When I fill Title and click Save
  Then the modal closes
  And the new task appears at the TOP of my task list
  And I see a toast: "Task added"

Edge Cases for Story 1:
  - If Title exceeds 200 chars → show error, prevent save
  - If Due Date is in past → show warning but allow (user can backdate)
  - If network fails on save → show error, keep modal open with data


## API Contracts (Frontend + Backend Alignment)
### Description of what to put here

### Examples
#### Bad

#### Good

## subTitle
### Description of what to put here
Specify every endpoint your frontend will call. Include request/response shapes, HTTP methods, and status codes.
Why this works: AI can generate the exact fetch call, error handling, and TypeScript types automatically.

### Examples
#### Bad
na

#### Good
Endpoint: POST /api/tasks
Authentication: Bearer token in header
Request Body:
  {
    title: string (required, 1-200 chars),
    description: string (optional),
    due_date: string (ISO date, optional)
  }
Success Response (201 Created):
  {
    task: { 
      id: string,
      user_id: string,
      title: string,
      description: string | null,
      status: 'pending',
      due_date: string | null,
      created_at: timestamp,
      updated_at: timestamp
    }
  }
Error Responses:
  400: { error: "Title is required" }
  401: { error: "Unauthorized" }
  409: { error: "Task title already exists for today" }

Rate Limit: 100 requests per minute per user


## State Management Strategy
### Description of what to put here
AI needs to know where data lives (local state? global store? URL params?) and how it updates.
Why this works: AI generates error boundaries, try/catch blocks, and retry logic automatically.

### Examples
#### Bad
na

#### Good
Global State (Zustand store):
  - user: User | null (null = logged out)
  - tasks: Task[] (cached for 5 minutes)
  - ui: { isLoading: boolean, error: string | null, filterStatus: string }

Local Component State:
  - TaskModal: { isOpen, titleInput, descriptionInput, dueDateInput }

URL State (React Router):
  - /dashboard?filter=completed → shows only completed tasks
  - /task/:id → opens detail view

Data Flow:
  1. User adds task via modal
  2. Optimistically update local tasks array (add temp id)
  3. POST to /api/tasks
  4a. If success: replace temp id with real id, show toast
  4b. If failure: revert tasks array, show error modal

## Edge Cases Checklist
### Description of what to put here
Most AI code breaks on edge cases. List them explicitly.

### Examples
#### Good
Empty States:
  - No tasks: Show illustration + "Create your first task" button
  - No search results: "No tasks match 'keyword'"
  - Empty filter: "No completed tasks"

Loading States:
  - Initial page load: Skeleton loaders for 3 tasks
  - Form submission: Disable submit button, show spinner
  - Pagination: Disable next/prev while fetching

Error States:
  - Network failure: "Check your connection. Retry?" button
  - Server error (500): "Something went wrong. Contact support."
  - Rate limit: "Too many requests. Wait 30 seconds."
  - Session expiry: Redirect to login with return URL

Data Conflicts:
  - User edits task that another user just deleted → show "Task no longer exists"
  - User submits form with stale data → show "Someone else updated this. Reload?"

## Tech Stack with Exact Versions
### Description of what to put here
Vague tech stack = AI uses wrong patterns. Be specific.
Why this works: AI fetches correct syntax from memory. It won't give you React class components or Redux boilerplate.

### Examples
#### Good
Frontend:
  - React 18.2.0 (functional components only, no classes)
  - TypeScript 5.0+ (strict mode on)
  - Vite 4.0 (not Create React App)
  - Tailwind CSS 3.3 (no custom CSS files)
  - React Router DOM 6.14 (using createBrowserRouter)
  - Zustand 4.3 (not Redux)
  - TanStack Query 4.29 (for server state)

Backend (if full-stack):
  - Node 20 LTS
  - Express 4.18
  - Prisma 5.0 (ORM)
  - PostgreSQL 15
  - Zod 3.21 (validation)
  - JSON Web Tokens for auth

Testing:
  - Vitest (unit/integration)
  - React Testing Library
  - MSW (mock service worker for API)

Conventions:
  - File naming: PascalCase for components, camelCase for utils
  - Folder structure: feature-based (not type-based)
  - Use path aliases: @/components, @/utils, @/stores

## Non-Negotiables (Hard Constraints)
### Description of what to put here
List things that are not optional—AI must include them.
Why this works: AI won't cut corners. It knows what to prioritize.

### Examples
#### Good
Accessibility (WCAG 2.1 AA):
  - All interactive elements must be keyboard-navigable
  - Images must have alt text
  - Color is not the only indicator (e.g., error state = icon + red text + aria-invalid)
  - Focus indicators visible on all buttons/inputs

Performance:
  - Lighthouse scores must be >90 on mobile emulation
  - No unnecessary re-renders (use memo, useCallback where needed)
  - Lazy-load modals and routes

Security:
  - No inline event handlers (onclick attributes)
  - Sanitize any user-generated content before rendering
  - CSRF tokens on all state-changing requests

Code Quality:
  - No `any` type in TypeScript
  - No `console.log` in production (use logger utility)
  - 80% test coverage on business logic

-----------

## How to write it (Practical Workflow)

### 1. Brain Dump
Write everything you know in bullet points. Ignore structure.

#### Example
- Users need accounts
- Tasks have titles and due dates
- Can mark complete
- Show overdue tasks in red
- Need search
- Maybe tags later but not now
- Should work on phone

### 2. Feed AI Brain Dump --> Structure
Suggested Prompt:
"Take this brain dump and organize it into the 7-part spec template I gave you (Data Models, UI Flows, API Contracts, State Management, Edge Cases, Tech Stack, Non-Negotiables). Ask me clarifying questions where information is missing."
AI will return a structured draft with [TODO] placeholders.

### 3. Fill in the Blanks
Answer AI's clarifying questions. For each [TODO], make a concrete decision:

#### Example
| Question | Weak Answer | Strong Answer |
|----------|-------------|---------------|
|"How many tasks per user?" | "I don't know, a lot" | "500 tasks max, paginate 20 at a time" |
|"What happens to completed tasks?" | "They stay" | "Show in separate 'Completed' tab, archive after 30 days" |
| "Due date timezone?" | "Use local time" | "Store UTC, display in user's browser timezone" |

### 4. Validate Spec with a Mini-Implementation (10 minutes)
Ask AI to implement just one endpoint or one component from the spec.
Suggested Prompt: 
"Based on the spec we just wrote, implement the POST /api/tasks endpoint using Express and Zod validation."
If AI's implementation contradicts your spec (e.g., missing rate limiting), your spec has ambiguity. Fix it.
#### Example
    Make a task manager. Users can add tasks with dates. Show tasks in a list. Use React."

AI's likely output:
- Uses localStorage (you wanted a backend)
- No loading states
- No due date validation
- Flat file structure with everything in App.js

After (AI-ready spec excerpt):
Data Model:
  Task: { id, title(1-200), due_date(ISO date, >= today), completed(boolean) }

UI Flow - Add Task:
  Given empty task list
  When I click "+" floating button
  Then modal opens with title input (auto-focus)
  When I type "Buy milk" and press Enter
  Then modal closes and task appears at top with "just now" badge

State Management:
  - React Query cache for tasks (key: ['tasks', userId])
  - Optimistic updates: add temp task immediately, revert on error

Tech Stack:
  - Vite + React 18
  - Tailwind + shadcn/ui components
  - React Hook Form + Zod for modal validation

Edge Cases:
  - Adding task with duplicate title → show warning but allow
  - Network slow > 2s → show "Still saving..." after 2 seconds

AI's output with this spec:
Generates proper optimistic updates, modal with form validation, loading states, and shadcn/ui components—all matching your exact stack

---
The 80/20 Rule for Spec Writing

You don't need every detail. Focus on the 20% that causes 80% of AI mistakes:
- Data validation rules (required fields, formats, lengths)
- State update patterns (optimistic vs pessimistic)
- Error recovery behavior (retry? revert? notify?)
- Tech stack specifics (library versions, file structure)
- Empty/loading states (most AI code forgets these)

Everything else (colors, animations, exact wording of buttons) AI can infer or you can tweak later.
---
Quick Reference: Spec Checklist Before Coding

Print this. Tick every box before giving spec to AI.

    Every data model has field types and constraints (min/max, required/optional)
    Every user story has Given-When-Then format with edge cases
    Every API endpoint has request/response shape + error codes
    State management specifies: optimistic or pessimistic updates? cache TTL?
    Edge cases: empty, loading, error, conflict, expiry
    Tech stack has exact versions (not "React" but "React 18.2 with Vite")
    Non-negotiables include accessibility and security baseline
    I have implemented ONE piece of the spec to confirm AI understands it
