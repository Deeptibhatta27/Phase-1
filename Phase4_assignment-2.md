# Phase 3 Day 3 — Assignment 2: Trace One Request End to End
**BetaninjasSEOLMS · app/api/progress/route.ts · prisma/schema.prisma · Deepti · June 2026**

---

## Step 1 — The Async Chain

The POST handler in `route.ts` contains exactly **3 await calls**. Reading them in order:

---

### Await 1 — `auth()`

```ts
const session = await auth()
```

**What is being awaited:** The `auth()` function from NextAuth. This reads the session cookie from the incoming request and verifies the JWT token to establish who is making the request.

**What type it returns:** A session object — something like `{ user: { id: string, name: string, email: string, role: string } }` — or `null` if no valid session exists.

**What happens if it returns null or throws:**
The very next line handles the null case explicitly:
```ts
if (!session?.user?.id) {
  return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
}
```
If `session` is `null`, or if `session.user` is undefined, or if `session.user.id` is missing, the handler immediately returns a 401 response and exits. No further code runs. If `auth()` throws an unexpected error (network failure, token library crash), it is **not** caught — it would propagate as an unhandled promise rejection and likely result in a 500 response from Next.js's default error handler.

---

### Await 2 — `request.json()`

```ts
const body = await request.json()
```

**What is being awaited:** Parsing the raw HTTP request body as JSON. This reads the body stream from the incoming POST request and deserialises it into a JavaScript object.

**What type it returns:** `any` — the type is unknown at this point. It is whatever the client sent, deserialised from JSON. It could be an object, an array, a string, or garbage.

**What happens if it returns null or throws:**
If the request body is not valid JSON (e.g. the client sent malformed JSON or an empty body), `request.json()` throws a `SyntaxError`. This is **not caught** in the handler. An unhandled throw here would result in a 500 error. The safe guard that follows is `progressSchema.safeParse(body)` — but that only runs if `request.json()` succeeds. If the body is completely unparseable, the error happens before Zod even sees it.

A defensive implementation would wrap this in a try/catch and return a 400 for malformed JSON. The current code does not do this.

---

### Await 3 — `prisma.progress.upsert(...)`

```ts
const progress = await prisma.progress.upsert({ ... })
```

**What is being awaited:** A database write operation via Prisma. `upsert` means: if a `Progress` row already exists for this `userId + taskId` combination, update it; if it does not exist, create it.

**What type it returns:** A `Progress` object — the full database row that was just written or updated, including `id`, `userId`, `taskId`, `status`, `response`, `link`, `completedAt`, `updatedAt`.

**What happens if it returns null or throws:**
Prisma `upsert` does not return null on success — it always returns the record. However it can throw in several scenarios:
- The database is unreachable (connection failure) → unhandled, results in 500
- A foreign key constraint fails (e.g. `taskId` does not exist in the `Task` table) → Prisma throws a `PrismaClientKnownRequestError`, also unhandled
- The unique constraint is violated in an unexpected way → unhandled

None of these DB-level errors are caught. If the database throws, the error propagates up and Next.js returns a 500 with no meaningful error message to the client. The member would see a generic failure with their input still in the field — which matches the spec's error state requirement, but not because the code handles it gracefully.

---

### Async Chain Summary

```
  POST /api/progress
       │
       ├── await auth()
       │     ├─ null / no id → return 401 ✓ (handled)
       │     └─ throws       → 500 ✗ (unhandled)
       │
       ├── await request.json()
       │     ├─ valid JSON   → body object (any)
       │     └─ throws       → 500 ✗ (unhandled — malformed body)
       │
       ├── progressSchema.safeParse(body)  [synchronous — not await]
       │     ├─ success      → parsed.data with typed fields
       │     └─ failure      → return 400 ✓ (handled)
       │
       └── await prisma.progress.upsert(...)
             ├─ success      → Progress record → return 200 ✓
             └─ throws       → 500 ✗ (unhandled — DB error)
```

---

## Step 2 — The Request

### What the Browser Sends (POST from a TEXT task submission)

**URL:** `https://betaninjas-seo-learning-tracker.vercel.app/api/progress`

**Method:** `POST`

**Headers:**
- `Content-Type: application/json` — tells the server the body is JSON
- `Cookie: next-auth.session-token=<JWT>` (or `__Secure-next-auth.session-token` on HTTPS) — this is **how the member proves they are logged in**. The session token is a signed JWT stored as an HTTP-only cookie. It travels automatically with every request to the same origin. The server never asks the client to send a user ID — it reads the identity from the verified session cookie.

**Request body (JSON):**
```json
{
  "taskId": 30,
  "status": "SUBMITTED",
  "response": "I completed keyword research using Ahrefs and identified 15 low-competition keywords for our blog."
}
```

| Field | Data Type | Required | Notes |
|---|---|---|---|
| `taskId` | `number` (integer, positive) | Yes | The ID of the task being submitted. Must match a real Task row in the DB. |
| `status` | `string` (enum) | Yes | One of: `NOT_STARTED`, `IN_PROGRESS`, `SUBMITTED`, `COMPLETED`. For a TEXT task submission, this is `SUBMITTED`. |
| `response` | `string` | Optional | The text the member typed. Only present for TEXT tasks. Omitted for CHECKBOX and LINK. |
| `link` | `string` (URL) | Optional | The URL the member submitted. Only present for LINK tasks. Omitted for TEXT and CHECKBOX. |

**How the member's identity travels:** The session cookie is sent automatically by the browser with every request to the same domain. The server calls `await auth()` which reads that cookie, verifies the JWT signature, and extracts `session.user.id`. The client never sends a user ID explicitly in the body — it cannot be trusted if it did. The only trusted identity is the server-verified session.

---

### What the Server Sends Back (on success)

**Status code:** `200 OK`

**Response body:** The full `Progress` record that was written to the database, serialised as JSON:

```json
{
  "id": 142,
  "userId": "clxyz123abc",
  "taskId": 30,
  "status": "SUBMITTED",
  "response": "I completed keyword research using Ahrefs and identified 15 low-competition keywords for our blog.",
  "link": null,
  "completedAt": null,
  "updatedAt": "2026-06-20T10:34:22.000Z"
}
```

The client receives the actual stored record — not a success message, not a boolean. This means the front-end can confirm exactly what was saved without making a separate GET request.

---

## Step 3 — The Database

### The `Progress` Model — Every Field

```prisma
model Progress {
  id          Int        @id @default(autoincrement())
  userId      String
  user        User       @relation(fields: [userId], references: [id])
  taskId      Int
  task        Task       @relation(fields: [taskId], references: [id])
  status      TaskStatus @default(NOT_STARTED)
  response    String?
  link        String?
  completedAt DateTime?
  updatedAt   DateTime   @updatedAt

  @@unique([userId, taskId])
}
```

| Field | Type | Required / Optional | Notes |
|---|---|---|---|
| `id` | `Int` | Required — auto-generated | Primary key. Assigned by the DB on insert. Never sent by the client. |
| `userId` | `String` | Required | Foreign key to `User.id`. The CUID of the member who owns this progress record. |
| `user` | Relation | — | Prisma relation field. Not a DB column — it is a join handle to the `User` model. |
| `taskId` | `Int` | Required | Foreign key to `Task.id`. Identifies which task this progress belongs to. |
| `task` | Relation | — | Prisma relation field. Join handle to the `Task` model. |
| `status` | `TaskStatus` (enum) | Required | Default: `NOT_STARTED`. One of: `NOT_STARTED`, `IN_PROGRESS`, `SUBMITTED`, `COMPLETED`. |
| `response` | `String?` | Optional | The member's written TEXT response. Null for CHECKBOX and LINK tasks. |
| `link` | `String?` | Optional | The URL submitted for a LINK task. Null for CHECKBOX and TEXT tasks. |
| `completedAt` | `DateTime?` | Optional | Timestamp set when status becomes `COMPLETED`. Null for all other statuses. |
| `updatedAt` | `DateTime` | Required — auto-managed | Automatically updated by Prisma every time the row is written. |

---

### The Unique Constraint — `@@unique([userId, taskId])`

```prisma
@@unique([userId, taskId])
```

**In plain English:** No two `Progress` rows can have the same combination of `userId` and `taskId`. One member can only have one progress record per task.

**What it prevents:** It prevents a member from accumulating multiple progress rows for the same task — which would make progress calculation ambiguous (which row counts?) and corrupt the percentage displayed. Without this constraint, submitting a task twice would create two rows, and `calcPhaseProgress` would either count the task twice or pick an arbitrary row.

**What it enables:** The `upsert` operation in `route.ts` relies on this constraint. Prisma's `upsert` uses `where: { userId_taskId: { userId, taskId } }` — meaning "find the row where this combination exists." If it finds one, it updates. If it does not, it creates. The unique constraint is what makes `upsert` work — without a unique index on the combination, Prisma cannot perform a lookup by both fields simultaneously.

**Practical consequence for the API:** A member submitting the same task a second time (revising their response) does not create a new row — it updates the existing one. The `response` field is overwritten, the `status` stays `SUBMITTED`, and `updatedAt` is refreshed. This is the "revision replaces old response" behaviour the spec requires, implemented entirely by the `upsert` + `@@unique` combination.

---

### `completedAt` — When It Is Set, When It Is Null

From the `upsert` code:

**On create:**
```ts
completedAt: status === "COMPLETED" ? new Date() : null,
```

**On update:**
```ts
...(status === "COMPLETED" && { completedAt: new Date() }),
...(status === "NOT_STARTED" && { completedAt: null }),
```

| Scenario | `completedAt` value |
|---|---|
| Task is checked off (CHECKBOX → COMPLETED) | Set to the current timestamp |
| Task is unchecked (COMPLETED → NOT_STARTED) | Explicitly reset to `null` |
| Task is submitted (TEXT/LINK → SUBMITTED) | Remains `null` — not set |
| Task is opened for first time (→ IN_PROGRESS) | Remains `null` |

**What it means for progress calculation:** `completedAt` is not used by `calcPhaseProgress` or `calcOverallProgress`. Those functions only look at `status` via `isDone()`. `completedAt` is a supplementary timestamp — it exists for audit, reporting, or future features (e.g. showing "you completed this on June 20th"). It is not part of the core progress calculation logic.

An important observation: `completedAt` is never set for `SUBMITTED` tasks, only for `COMPLETED` ones. Since `SUBMITTED` is the terminal state for TEXT and LINK tasks, those tasks will always have `completedAt = null` in the database — even when they are fully done and counting toward 100%. This means `completedAt` cannot be used as a reliable "task is done" signal across all task types. `status` is the authoritative field.

---

### Relation Chain Diagram

```
  Progress ──────────→ Task ──────────→ Phase
  │                    │                │
  userId (FK)          phaseId (FK)     id (PK)
  taskId (FK)          id (PK)          title
  status               title            order
  response             type             tasks[]
  link                 order            timeline?
  completedAt          required
  │
  └──────────→ User
               id (PK)
               name
               email
               role
```

**Reading this:** One `Progress` record belongs to exactly one `User` (the member who owns it) and exactly one `Task` (the task it tracks). That `Task` belongs to exactly one `Phase`. So the full chain for a single progress record is: **Progress → Task → Phase** and **Progress → User**. The `@@unique([userId, taskId])` constraint means the intersection of User and Task is unique — each member has at most one progress record per task.

---

## Step 4 — What the UI Shows vs What the Database Stores

---

### Q1: "3 of 8 tasks" — stored or calculated?

**Calculated every time. It is never stored.**

The database stores individual `Progress` rows — one per member per task, each with a `status` field. The numbers "3" and "8" do not exist anywhere in the database as columns or values.

When the dashboard or phase page loads, the server fetches all tasks for a phase and all progress records for the current user. It then calls `calcPhaseProgress(tasks, progressMap)`, which counts how many task IDs have a status of `SUBMITTED` or `COMPLETED` in the map. That count becomes the numerator. `tasks.length` is the denominator. The resulting percentage — and by extension the "3 of 8" display — is computed fresh on every page load.

**Implication for testing:** If `calcPhaseProgress` has a bug, every member sees wrong numbers on every page load. There is no stored value to fall back on. This is why the function needs unit tests — not because the calculation is complex, but because it is the single source of truth for every number the product displays.

---

### Q2: Member submits a TEXT task. UI shows "Submitted." What is stored in the database?

The `status` column in the `Progress` row is set to **`SUBMITTED`** — the exact enum value from `TaskStatus`. The `response` column stores the member's text exactly as it was received by the server. `completedAt` remains `null`. `link` remains `null`.

The UI label "Submitted." is a client-side rendering decision — the front-end reads the `status` value and maps `"SUBMITTED"` to the display string "Submitted." They are not the same string. The DB stores `"SUBMITTED"` (uppercase, no punctuation); the UI shows `"Submitted."` (title case, with a period).

---

### Q3: Member checks a CHECKBOX task. UI shows a green tick. What is stored? What additional field is written?

The `status` column is set to **`COMPLETED`**.

The additional field written is **`completedAt`** — set to `new Date()`, the timestamp of the moment the checkbox was checked. This is the only task type and the only status transition that triggers `completedAt` being set.

`response` and `link` remain `null` — CHECKBOX tasks have no text or URL submission.

If the member unchecks the task (toggling back), the `upsert` fires again with `status: "NOT_STARTED"` and `completedAt: null` — explicitly clearing the timestamp. The row is not deleted; it is updated in place.

---

### Q4: Phase card shows "ON TRACK". Is this stored in the database?

**No. It is not stored anywhere in the database.**

`"ON_TRACK"` is the return value of `getPhaseStatus()` — a pure utility function in `lib/utils/timeline.ts` that takes `progressPercent`, `startDate`, `dueDate`, and `now` as inputs and computes a status label. It is called at render time, every time the dashboard loads.

The database stores:
- Individual `Progress` rows (from which `progressPercent` is calculated by `calcPhaseProgress`)
- `PhaseTimeline` rows with `startDate` and `dueDate`

`getPhaseStatus` combines those two data points with the current timestamp and returns one of five string labels. The label `"ON TRACK"` displayed in the UI is the result of that runtime calculation — it is never written to any table, never cached, and does not appear in the Prisma schema. If the timeline changes or the member completes more tasks, the next page load recalculates it automatically.

---

### Q5: Is there any transformation between what the member typed and what is stored?

**No transformation. The text is stored exactly as received.**

Tracing the path: the member's typed text travels as a JSON string in the request body → `request.json()` deserialises it → `progressSchema.safeParse(body)` validates it as `z.string().optional()` — Zod does not trim, sanitise, or modify string values by default → `parsed.data.response` holds the exact string → it is passed directly into the Prisma `upsert` as `response: response ?? null`.

There is no `trim()`, no HTML escaping, no length truncation, no encoding transformation in the route handler or the schema.

**What this means for testing:**
- A response with leading/trailing spaces is stored with those spaces
- A response with HTML tags like `<script>alert('xss')</script>` is stored as a raw string — the database itself does not execute it, but if the front-end renders this value as `innerHTML` rather than `textContent`, it could be a stored XSS vector
- A very long response is stored in full — there is no server-side length limit enforced in this handler (connecting back to the BVA-T5 finding from Phase 2 about the missing upper bound)

The lack of transformation is both a simplicity strength (what you type is what you get back) and a potential security and data integrity risk (no sanitisation, no length enforcement).

---

*Submitted by Deepti · Phase 3 Day 3 · Assignment 2 · June 2026*
