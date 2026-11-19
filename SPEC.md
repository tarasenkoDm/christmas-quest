# 🎄 Christmas Quest — Specification

A step-by-step quest game. Players log in, press **Start**, and solve a sequence of tasks. Each task has a countdown; when it expires, hints are revealed one by one, and finally the answer. Admins manage tasks and users.

---

## 1. Tech stack & architecture

- **Angular 22**: standalone components, signals, new control flow (`@if` / `@for`), reactive forms (login + admin forms).
- **Bootstrap 5 + SCSS** (existing SCSS reset), with a Christmas theme.
- **Vitest** for unit tests. No E2E tests.
- **Client-only persistence**: signal-based services backed by `localStorage`, exposed through repository interfaces (`UserRepository`, `TaskRepository`, `ProgressRepository`) so a real backend can be swapped in later.
- Admin routes are **lazy-loaded**.
- Follow the conventions in `.claude/skills`.
- Deployed as a static site on Vercel.

> ⚠️ Auth is demo-grade: passwords are stored and displayed in plain text in the browser.

---

## 2. Data models

### 2.1 User
| Field      | Type                  | Rules                                                   |
|------------|-----------------------|---------------------------------------------------------|
| `id`       | `number`              | auto-generated                                          |
| `username` | `string`              | required, min 3 chars, **unique (case-sensitive)**      |
| `password` | `string`              | required, min 4 chars, plain text                       |
| `role`     | `'admin' \| 'user'`   | required                                                |

- **Guest** is not stored — it is the state of "nobody is logged in".

### 2.2 Task
| Field            | Type                                   | Rules                                             |
|------------------|----------------------------------------|---------------------------------------------------|
| `id`             | `number`                               | auto-generated                                    |
| `title`          | `string`                               | required                                          |
| `description`    | `string`                               | required                                          |
| `answers`        | `string[]`                             | at least 1; accepted alternatives                 |
| `timerMinutes`   | `number`                               | base task timer, **≥ 0.01**, decimals allowed     |
| `hints`          | `{ text: string; timerMinutes: number }[]` | 0–3 items; each `text` required, `timerMinutes` **≥ 0.01** |
| `assignedUserId` | `number \| null`                       | single user; `null` = **draft** (nobody plays it) |
| `order`          | `number`                               | positive integer; defines play sequence           |

### 2.3 Progress (per user)
| Field         | Type             | Meaning                                                      |
|---------------|------------------|--------------------------------------------------------------|
| `userId`      | `number`         | owner                                                        |
| `currentStep` | `number`         | index of the current task (1-based in the URL)               |
| `startedAt`   | `string \| null` | ISO timestamp saved when the player clicks the big **Start** |
| `finishedAt`  | `string \| null` | ISO timestamp saved when the **Quest complete** page is reached |

### 2.4 Session
- The logged-in user's `id` is stored in `localStorage`.

### 2.5 Seed data
- Accounts: `admin / admin` (admin), `alice / 1234` (user), `bob / 1234` (user).
- A few Christmas-themed sample tasks assigned to `alice` and `bob` (and some to `admin`), plus at least one draft.
- Seed accounts are listed in the README.

---

## 3. Authentication & session

- Login form on the welcome page: **username** + **password**.
- Matching is **case-sensitive** for both fields.
- Failed login → generic message *"Invalid username or password"* under the form, both fields get a red border.
- Session persists across reloads/tab reopen (`localStorage`).
- On app start the stored session is **re-validated** against the user store:
  - user deleted → becomes guest;
  - role changed → new role applies immediately.
- **Header (top-right)** when logged in: username + **Sign Out**. Admins additionally see **Manage tasks** and **Manage users** links.
- **Sign Out** → clears session, status becomes guest, navigates to `/`.

---

## 4. Roles & access

| Role  | Can do                                                                 |
|-------|------------------------------------------------------------------------|
| guest | view the welcome page only                                             |
| user  | play their assigned tasks                                              |
| admin | play their assigned tasks + full CRUD on tasks and users               |

---

## 5. Routes

| Route                       | Access       | Purpose                                   |
|-----------------------------|--------------|-------------------------------------------|
| `/`                         | everyone     | welcome page                              |
| `/quest/:step`              | user, admin  | task page                                 |
| `/quest/complete`           | user, admin  | quest complete page                       |
| `/admin/tasks`              | admin        | task list                                 |
| `/admin/tasks/new`          | admin        | create task                               |
| `/admin/tasks/:id`          | admin        | task detail (read-only)                   |
| `/admin/tasks/:id/edit`     | admin        | edit task                                 |
| `/admin/users`              | admin        | user list                                 |
| `/admin/users/new`          | admin        | create user                               |
| `/admin/users/:id`          | admin        | user detail (read-only)                   |
| `/admin/users/:id/edit`     | admin        | edit user                                 |

- Unauthorized access (guest on quest/admin, user on admin) → silent redirect to `/`.
- **No skipping, no going back**: any `/quest/:step` other than the player's current step redirects to the current step.

---

## 6. Welcome page

- App description.
- **Guest**: login form.
- **Logged in**: login form hidden, shows *"Welcome, {name}!"* and a big button:
  - **Start** — no progress yet. Clicking it **saves `startedAt` = now** in storage and goes to `/quest/1`.
  - **Continue** — progress exists. Resumes at the current step; `startedAt` is **not** changed.
  - If the player has **zero assigned tasks**: *"No quest tasks have been assigned to you yet."* instead of the button.

---

## 7. Gameplay (task page)

### 7.1 Task set & order
- A player gets only tasks where `assignedUserId === currentUser.id`, sorted by `order` ascending.
- Shows **"Task N of M"**, the task **title** and **description**.

### 7.2 Timer & hint stages
```
task timer → Hint 1 shown + hint-1 timer → Hint 2 … → Hint 3 timer → answer revealed, "Time's up"
```
- Tasks with fewer than 3 hints skip the missing stages (0 hints → task timer expiry reveals the answer).
- Display: large `MM:SS` countdown, stage label (*"Time until hint 1"*, *"…hint 2"*, *"…hint 3"*, *"…answer reveal"*, *"Time's up"*), and a shrinking Bootstrap progress bar.
- Timer is **wall-clock based**: stays accurate in background tabs; stages that elapsed while hidden are shown on return.
- Revealed hints stack under the description as Bootstrap alert cards (*Hint 1/2/3*) with a fade-in. All stay visible.
- After the last timer: timer shows `00:00` / *"Time's up"*, all hints visible, and the **correct answer (`answers[0]`)** is shown in a highlighted card. The player must still **type it and click Answer** to proceed.

### 7.3 Answering
- Text input + **Answer** button. **Enter** also submits.
- Empty / whitespace-only input → button disabled, not counted as wrong.
- Matching: trim + **case-insensitive** against every entry in `answers` (`"Santa "` matches `"santa"`).

### 7.4 Correct answer
- Full-screen overlay with a cheerful spinner animation (spinning 🎄/star, *"Correct! 🎉"*, CSS confetti) for ~2 s. Timer pauses.
- Progress advances; navigates to the next task where timers restart from the beginning.
- After the last task → `/quest/complete`.

### 7.5 Wrong answer
- Full-screen "sad" overlay (😢 shake/droop, *"Not quite…"*) for ~1.5 s, then hides automatically.
- **Timer keeps running.**
- Input gets a red border; it clears as soon as the player starts typing.

### 7.6 Resume
- Progress (current step) is saved per user. Returning (reload, sign out/in) resumes at that task, but **its timer and hints restart** from the beginning.

---

## 8. Quest complete page

- On first arrival **saves `finishedAt` = now** in storage (not overwritten on revisits/reloads).
- Shows a celebratory animation, *"Quest complete 🎄"*, and **"You spent XX minutes"** = `finishedAt − startedAt`.
  - Display format: whole minutes; if under 1 minute, show seconds instead (e.g. *"You spent 12 minutes"*, *"You spent 45 seconds"*).
- **Play again** button → resets progress (`currentStep`, `startedAt`, `finishedAt` cleared) and returns to the welcome page, where **Start** is shown again.

---

## 9. Admin — task management

### 9.1 List (`/admin/tasks`)
Table columns:

`id` · `order` · `title` · `description` · `assigned user` (by name) · `timer` · `hint1` · `hint2` · `hint3` · `count1` · `count2` · `count3` · `answer` · `actions (View / Edit / Delete)`

- `timer` = base task timer (min); `count1–3` = hint timers (min).
- Long texts (description, hints) truncated to ~50 chars with "…".
- Answers shown comma-separated.
- **Sort**: the **Order** header is clickable; click again reverses direction. Drafts are always grouped at the end.
- **Filter**: dropdown above the table — *All* / each user / *Drafts*.
- **Add task** button → `/admin/tasks/new`.
- Table scrolls horizontally on small screens.

### 9.2 Detail (`/admin/tasks/:id`)
- Read-only view of all fields (full text).

### 9.3 Create / Edit form
- `title` — required
- `description` — required
- `answers` — list editor, at least 1
- `timerMinutes` — required, ≥ 0.01
- `hints` — add/remove buttons, max 3; each with `text` (required) and `timerMinutes` (≥ 0.01)
- `assignedUserId` — dropdown of users + *"Unassigned (draft)"*
- `order` — positive integer

### 9.4 Delete
- Bootstrap confirmation modal.

---

## 10. Admin — user management

### 10.1 List (`/admin/users`)
Table columns: `id` · `username` · `password` · `role` · `actions (View / Edit / Delete)`

- **Add user** button → `/admin/users/new`.

### 10.2 Detail / Create / Edit
- `username` — required, min 3 chars, unique (case-sensitive)
- `password` — required, min 4 chars
- `role` — required, `admin` or `user`

### 10.3 Rules
- Delete → Bootstrap confirmation modal.
- Deleting a user → their tasks become **drafts** (`assignedUserId = null`) and their progress is cleared.
- An admin **cannot delete or demote themselves**; the **last admin** cannot be removed/demoted.
- Editing the logged-in user (e.g. rename) updates the header immediately.

---

## 11. UI / UX

- Christmas theme on Bootstrap: deep red / green / gold palette, snowfall background effect, festive heading font.
- English only (no i18n).
- Responsive / mobile-friendly.
- Animations are pure CSS / Angular animations (no extra libraries) and respect `prefers-reduced-motion`.

---

## 12. Testing (Vitest unit tests)

- Answer matching (trim, case-insensitive, multiple answers).
- Timer / hint stage state machine (including 0–3 hints, background-tab catch-up).
- Auth service and route guards (guest / user / admin, session re-validation).
- Store operations: task CRUD, user CRUD, uniqueness, delete-user cascade, last-admin / self protection.
- Progress: `startedAt` on Start, `finishedAt` on complete, elapsed-time formatting, Play again reset.
- Admin task list sort and filter.
