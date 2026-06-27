# Day 5 — Assignment 1: Map the Coverage
**BetaninjasSEOLMS · Deepti · June 2026**

---

## Step 1 — The Testing Pyramid

### All Test Cases from Phase 1 and Phase 2 — Categorised by Level

Going through every test case written across Phase 1 (Login feature — Black Box, Grey Box, White Box) and Phase 2 (BVA, Equivalence Partitioning, Decision Table, State Transition, Testing Levels) and placing each one at its correct pyramid level.

---

#### Phase 1 — Login Feature

| TC ID | Test Case | Level | Reason |
|---|---|---|---|
| TC-01 | Wrong password shows error, user stays on login page | **System** | Full browser → UI → server → response cycle. No isolation. |
| TC-02 | Empty fields show validation errors | **System** | End-to-end form validation through the rendered UI. |
| TC-03 | Valid credentials redirect to dashboard | **System** | Full auth flow: UI → NextAuth → session → redirect. |
| TC-04 | Invalid email format shows inline error | **System** | Client-side validation observed through the live UI. |
| TC-05 | 500-char input handled gracefully | **System** | Stress input through the full request cycle, observed in-browser. |
| GBC-01 | Non-admin navigating to `/admin` is redirected | **Integration** | Tests middleware + auth session working together — two layers interacting, not the full UI flow. |
| GBC-02 | Logged-in user revisiting `/login` is redirected | **Integration** | Tests middleware logic against the existing session state — integration of auth and routing layers. |
| WB-01 | `getDaysRemaining` — due today, slightly later | **Unit** | One isolated function, specific inputs, no UI, no DB. |
| WB-02 | `getDaysRemaining` — due date in the past | **Unit** | Same isolated function, negative diff path. |
| WB-03 | `getDaysRemaining` — due date exactly now | **Unit** | Same function, zero-diff edge case. |

#### Phase 2 — Task Submission & Timeline

| TC ID | Test Case | Level | Reason |
|---|---|---|---|
| BVA-Date (all 8) | Start/Due date boundary cases on admin timeline | **System** | Tested through the live `/admin/timeline` UI — involves rendering, form submit, and DB save. |
| BVA-T1 | Empty TEXT field — submit disabled | **System** | Observed via the live task page UI. |
| BVA-T2 | 1 character in TEXT field — submit enabled | **System** | Live UI interaction through the browser. |
| BVA-T3 | 255 characters saved correctly | **System** | Full end-to-end: type, submit, refresh, verify persistence. |
| BVA-T4 | 256 characters — no silent truncation | **System** | Tests the DB boundary through the full submission flow. |
| BVA-T5 | 5000-char paste — no server error | **System** | Stress test through the full stack. |
| EP-01 | CHECKBOX toggle → Completed | **System** | Full UI flow — click, state change, progress update observed. |
| EP-02 | Valid TEXT submit | **System** | Full submission flow through the UI. |
| EP-03 | Valid LINK submit, persists on refresh | **System** | Includes persistence check — full stack (UI → API → DB → reload). |
| EP-04 | Empty TEXT blocked | **System** | Client-side enforcement observed through the UI. |
| EP-05 | Empty LINK blocked | **System** | Client-side enforcement observed through the UI. |
| EP-06 | `https://,` malformed URL — should block (Known Bug) | **System** | End-to-end: type, observe field validation, click submit, observe state. |
| EP-07 | Plain text in URL field — blocked | **System** | Live UI validation test. |
| EP-08 | `ftp://` protocol — blocked | **System** | Live UI validation test. |
| EP-09 | CHECKBOX task has no textarea | **System** | UI rendering check — correct component shown for task type. |
| EP-10 | Single-char TEXT accepted | **System** | Full submit flow through the UI. |
| DT Row 1 | Unauthenticated POST → 401 | **Integration** | API-level test — session layer + route handler. No UI involved. |
| DT Row 2 | Invalid taskId → 404 | **Integration** | API + DB lookup interaction — two layers. |
| DT Row 3 | Invalid status enum → 400 | **Integration** | API validation layer — route handler + input parsing. |
| DT Row 4 | Empty text POST → 400 | **Integration** | API server-side validation — tests the server layer independently of the UI. |
| DT Row 5 | All valid → 200, saved | **Integration** | API + DB write + response — integration of route, DB, and session. |
| ST-01 | CHECKBOX NOT_STARTED → COMPLETED | **System** | Full UI click → state change → progress update. |
| ST-02 | CHECKBOX COMPLETED → NOT_STARTED | **System** | Full UI toggle. |
| ST-03 | TEXT auto-moves to IN_PROGRESS on open | **System** | UI + API interaction on task open — full flow. |
| ST-04 | TEXT IN_PROGRESS → SUBMITTED | **System** | Full submission through the UI. |
| ST-05 | TEXT SUBMITTED → revise → SUBMITTED | **System** | Full revise + resubmit flow through the UI. |
| ST-06 | Skip NOT_STARTED → SUBMITTED via API | **Integration** | Direct API call, testing server-side state transition enforcement. |
| ST-07 | TEXT taskType + COMPLETED status via API | **Integration** | Direct API call, testing type-state compatibility enforcement. |

---

### Count by Level

| Level | Count | % of total |
|---|---|---|
| Unit | 3 | ~7% |
| Integration | 9 | ~22% |
| System | 31 | ~71% |
| Acceptance | 0 | 0% |
| **Total** | **43** | **100%** |

---

### Shape — What Does This Look Like?

```
         Acceptance  ░░  (0 tests)
        ─────────────────────────────
       Unit      ███  (3 tests)
      ──────────────────────────────────
     Integration  █████████  (9 tests)
    ────────────────────────────────────────
   System    ███████████████████████████████  (31 tests)
  ──────────────────────────────────────────────────────
```

This is an **inverted pyramid** — the widest layer is at the top (System), and it narrows as you go down toward Unit. This is the opposite of what good test architecture looks like.

The shape makes sense given how the assignments were structured: we were testing a live app through the browser, which naturally produces System-level observations. But it means the test suite is slow, brittle, and expensive to maintain — System tests depend on the full stack being available, they break when UI changes, and they give no signal about *where* in the code a failure lives.

---

### What the Ideal Pyramid Would Look Like for BetaninjasSEOLMS

```
        ┌──────────────────────────────────────┐
        │        Acceptance (3–5 tests)         │  ← Real user completes a phase
        ├──────────────────────────────────────┤
        │        System (8–12 tests)            │  ← Full login-to-submit flows
        ├──────────────────────────────────────┤
        │        Integration (12–18 tests)      │  ← API + DB + auth layer
        ├──────────────────────────────────────┤
        │        Unit (25–40 tests)             │  ← Functions, validators, utils
        └──────────────────────────────────────┘
```

#### What Is Missing at Each Level

**Unit — severely under-tested**
Currently only 3 unit tests exist, all on `getDaysRemaining`. Missing:
- Tests for `getPhaseStatus` (4 branches, 6 paths — see Step 2)
- Tests for `calcPhaseProgress` and `calcOverallProgress` (progress calculation logic is critical — a bug here breaks every number on the dashboard)
- Tests for `isPhaseUnlocked` (the lock/unlock logic for phase progression)
- Tests for URL validation logic — the `https://,` bug (BUG-001) would have been caught here with a single unit test

**Integration — partially covered, but only via direct API calls in test cases**
The 9 integration-level tests were written as manual steps (POST via API tool), not as automated integration tests. Missing:
- Automated tests for the `/api/submit` route with real DB transactions
- Auth middleware integration — session-present and session-absent scenarios
- Progress recalculation after submission — does the DB update trigger the right recomputed value?

**System — over-represented but not the wrong tests to have**
The 31 system tests are real and useful. The problem is they are the *only* tests. They should be reduced to the 8–12 most critical happy paths and key error flows. The rest should be pushed down to unit or integration level where they run faster and fail with more useful messages.

**Acceptance — completely missing**
Zero acceptance tests exist. For BetaninjasSEOLMS specifically, acceptance testing matters because the product's value is in the learning experience, not just the data plumbing. An acceptance test for this app looks like: a real BetaNinjas team member sits down, completes 3 tasks across two phases without any guidance, and confirms that their progress reflects what they did and their submitted text is exactly what they typed. No QA engineer running a scripted test catches the assumption gaps that acceptance testing surfaces.

---

## Step 2 — Line, Branch, and Path Coverage

### The Two Functions from `timeline.ts`

The assignment points at `getDaysRemaining` but notes to "refer: getPhaseStatus". Both functions are in the same file. `getDaysRemaining` is 2 executable lines — it is the simpler example to explain the concepts. `getPhaseStatus` is the richer function with real branching. This step analyses both, starting with the simpler one to build the framework, then applying it to `getPhaseStatus`.

---

### `getDaysRemaining` — The Simple Case

```ts
export function getDaysRemaining(dueDate: Date, now: Date = new Date()): number {
  const diff = dueDate.getTime() - now.getTime()        // Line 1
  return Math.ceil(diff / (1000 * 60 * 60 * 24))        // Line 2
}
```

#### Line Coverage

One test calling `getDaysRemaining(futureDate, now)` with any valid future date executes:
- Line 1 ✅ — `diff` is calculated
- Line 2 ✅ — `Math.ceil` is applied and the result is returned

**Result: 1 test achieves 100% line coverage on `getDaysRemaining`.** Both lines are hit. There are no unreachable lines.

```
  getDaysRemaining(dueDate, now)
  │
  ├── Line 1: const diff = ...        ← always executed
  └── Line 2: return Math.ceil(...)   ← always executed
```

#### Branch Coverage

`getDaysRemaining` has **zero explicit if/else branches**. There is no conditional logic — the function always executes both lines regardless of input.

However, there is an **implicit branch** hidden in `Math.ceil`:
- `diff > 0` → positive result (future date)
- `diff = 0` → returns 0 (due right now)
- `diff < 0` → negative result (overdue)

These are not code branches — `Math.ceil` handles all three internally without an `if` statement. For formal branch coverage of the written code, **1 test is sufficient** because there are no branches to take.

For *meaningful* test coverage of the function's behaviour, **3 tests** are needed — one per sign of `diff` — to verify that the function behaves correctly in all real-world scenarios. But this is about test completeness, not branch coverage as a metric.

**Formal branch coverage: 1 test. Meaningful behaviour coverage: 3 tests.**

#### Path Coverage

Because `getDaysRemaining` has no conditionals, there is exactly **1 path** through the function: Line 1 → Line 2 → return.

**1 test achieves 100% path coverage.**

Path coverage equals line coverage equals branch coverage for this function because there is nothing to branch on. It is a straight line from entry to exit.

---

### `getPhaseStatus` — The Real Coverage Challenge

```ts
export function getPhaseStatus(
  progressPercent: number,
  startDate: Date,
  dueDate: Date,
  now: Date = new Date()
): TimelineStatus {
  if (progressPercent === 100) return 'COMPLETED'              // Branch A
  if (now < startDate) return 'NOT_STARTED'                    // Branch B
  if (now > dueDate && progressPercent < 100) return 'OVERDUE' // Branch C (compound)

  const totalDuration = dueDate.getTime() - startDate.getTime()  // Line 4
  const elapsed = now.getTime() - startDate.getTime()             // Line 5
  const timeElapsedPercent = Math.min((elapsed / totalDuration) * 100, 100)  // Line 6

  return progressPercent >= timeElapsedPercent ? 'ON_TRACK' : 'BEHIND'  // Branch D
}
```

#### Line Coverage

Call the function once where it reaches the bottom (ON_TRACK or BEHIND). For example:
`getPhaseStatus(50, startDate, dueDate, now)` where `now` is between `startDate` and `dueDate`.

Lines executed:
- Branch A check: `progressPercent === 100` → false, does not return ✅
- Branch B check: `now < startDate` → false, does not return ✅
- Branch C check: `now > dueDate` → false, does not return ✅
- Line 4: `totalDuration` calculated ✅
- Line 5: `elapsed` calculated ✅
- Line 6: `timeElapsedPercent` calculated ✅
- Branch D: ternary evaluated, returns ON_TRACK or BEHIND ✅

**Lines NOT hit by this single test:**
- The `return 'COMPLETED'` line inside Branch A ❌
- The `return 'NOT_STARTED'` line inside Branch B ❌
- The `return 'OVERDUE'` line inside Branch C ❌

**Conclusion: 1 test gets you to ~57% line coverage (4 of 7 executable outcomes hit). To achieve 100% line coverage you need at least 4 tests** — one that triggers each early return (A, B, C) and one that reaches the bottom (D).

```
  Minimum tests for 100% line coverage:

  Test 1: progressPercent = 100              → hits return 'COMPLETED'     (Line A)
  Test 2: now < startDate                    → hits return 'NOT_STARTED'   (Line B)
  Test 3: now > dueDate, progress < 100      → hits return 'OVERDUE'       (Line C)
  Test 4: now between dates, progress = 50   → hits Lines 4–6 + Branch D   (Lines 4,5,6,D)
```

#### Branch Coverage

Branches in `getPhaseStatus`:

| Branch | Condition | True outcome | False outcome |
|---|---|---|---|
| A | `progressPercent === 100` | return COMPLETED | fall through to B |
| B | `now < startDate` | return NOT_STARTED | fall through to C |
| C | `now > dueDate && progressPercent < 100` | return OVERDUE | fall through to lines 4–6 |
| C sub-condition | `now > dueDate` AND `progressPercent < 100` | Both must be true | Either false = fall through |
| D | `progressPercent >= timeElapsedPercent` | return ON_TRACK | return BEHIND |

**How many tests for 100% branch coverage?**

Every branch must be taken in both the true and false direction at least once.

- Branch A true: `progressPercent = 100` → COMPLETED
- Branch A false: any `progressPercent < 100` (covered by any other test)
- Branch B true: `now < startDate` → NOT_STARTED
- Branch B false: `now >= startDate` (covered by tests that reach C or D)
- Branch C true: `now > dueDate && progressPercent < 100` → OVERDUE
- Branch C false: `now <= dueDate` (covered by tests that reach D)
- Branch D true: `progressPercent >= timeElapsedPercent` → ON_TRACK
- Branch D false: `progressPercent < timeElapsedPercent` → BEHIND

**Minimum tests for 100% branch coverage: 5**

```
  Test 1: progressPercent = 100                          → Branch A true  → COMPLETED
  Test 2: now < startDate, progress = 0                  → Branch A false, Branch B true  → NOT_STARTED
  Test 3: now > dueDate, progress = 50                   → A false, B false, Branch C true → OVERDUE
  Test 4: now = midpoint, progress ahead of time         → A,B,C false, Branch D true → ON_TRACK
  Test 5: now = midpoint, progress behind time           → A,B,C false, Branch D false → BEHIND
```

Note on Branch C's compound condition (`now > dueDate && progressPercent < 100`): since `progressPercent === 100` is already caught by Branch A, any test reaching Branch C with `now > dueDate` will always have `progressPercent < 100`. The second sub-condition cannot independently be false at this point — it is logically guaranteed by Branch A. For strict MC/DC (Modified Condition/Decision Coverage), you would need to isolate each sub-condition — but for standard branch coverage, 5 tests is sufficient.

#### Path Coverage

A path is one complete route from the function's entry to its exit. Every combination of branch outcomes is a distinct path.

Paths through `getPhaseStatus`:

```
  Path 1: A=true                              → return COMPLETED
  Path 2: A=false → B=true                   → return NOT_STARTED
  Path 3: A=false → B=false → C=true         → return OVERDUE
  Path 4: A=false → B=false → C=false → D=true  → return ON_TRACK
  Path 5: A=false → B=false → C=false → D=false → return BEHIND
```

**Total paths: 5. Tests needed for 100% path coverage: 5.**

In this function, path coverage and branch coverage require the same number of tests (5) because the branches are sequential (not nested). There are no loops and no branches inside branches, so every unique path corresponds directly to a unique branch combination.

> **Important note:** Path coverage gets exponentially more expensive than branch coverage when branches are *nested*. If Branch D were *inside* Branch C (i.e. if you only evaluated ON_TRACK vs BEHIND when overdue), the path count would multiply. For a function with `n` nested `if/else` pairs, branch coverage needs `2n` tests but path coverage needs `2^n`. That is why path coverage is rarely used on complex functions — the cost is prohibitive.

---

### Line vs Branch vs Path Coverage — In Your Own Words, Using This File

**Line coverage** asks: did a test touch this line of code at all?

In `getDaysRemaining`, one test hits both lines. In `getPhaseStatus`, one test only hits 4 of the 7 outcome lines — the three early-return lines are invisible to it. Line coverage is the weakest metric. You can hit every line with one test per line and still miss half the function's behaviour if you never take the `false` side of a condition.

**Branch coverage** asks: for every `if`, did a test go both ways — true *and* false?

In `getDaysRemaining` there are no branches, so line = branch coverage. In `getPhaseStatus`, you need 5 tests to take every branch in both directions. Branch coverage is stronger than line coverage because it forces you to test what happens when conditions are *not* met — not just when they are. A test that only ever hits the `true` path of an `if` gives you line coverage but not branch coverage.

**Path coverage** asks: for every possible route from start to finish, did a test walk exactly that route?

In `getPhaseStatus`, all 5 paths happen to be the same as the 5 branch combinations because the branches are sequential, not nested. Path coverage costs the same here. But the moment you add a nested `if` inside one of the branches, the path count explodes: 2 nested branches = 4 paths, 3 nested = 8, and so on. Path coverage is theoretically complete but practically unachievable on any real-world function with loops or deep nesting. It is the most thorough and the most expensive.

**The practical takeaway for BetaninjasSEOLMS:**

`getDaysRemaining` is fully covered by 1 test for all three metrics — it is too simple to differentiate them. `getPhaseStatus` is the interesting case: currently it has zero unit tests, which means 0% line, 0% branch, and 0% path coverage on a function that drives every status badge visible to every member on every phase card. The risk is real. Five unit tests — one per path — would give complete coverage at all three levels simultaneously.

---


---

## Step 3 — Function Coverage

### All Utility Functions Found in `lib/`

From the four files provided (`lib/utils/progress.ts`, `lib/utils/timeline.ts`, `lib/validations/progress.ts`, `lib/validations/plan.ts`) plus the `timeline.ts` utils file from Phase 1:

| File | Function / Export | What It Does |
|---|---|---|
| `lib/utils/progress.ts` | `isDone(taskId, progressMap)` | Returns true if a task's status is SUBMITTED or COMPLETED. Internal helper — not exported. |
| `lib/utils/progress.ts` | `calcPhaseProgress(tasks, progressMap)` | Calculates % of tasks done in a phase. Returns 0 for empty task list. Rounds to nearest whole number. |
| `lib/utils/progress.ts` | `calcOverallProgress(phases, progressMap)` | Flattens all tasks across all phases, calculates overall % done. |
| `lib/utils/progress.ts` | `isPhaseUnlocked(phaseOrder, phases, progressMap)` | Determines whether a phase is accessible. Currently hardcoded to `return true` — real logic is commented out. |
| `lib/utils/timeline.ts` | `getPhaseStatus(progressPercent, startDate, dueDate, now)` | Returns one of 5 status labels: NOT_STARTED, ON_TRACK, BEHIND, COMPLETED, OVERDUE. |
| `lib/utils/timeline.ts` | `getDaysRemaining(dueDate, now)` | Calculates days remaining until a due date. Returns negative if overdue. |
| `lib/validations/progress.ts` | `progressSchema` (Zod schema) | Validates task submission input: taskId (int+), status (enum), optional response (string), optional link (url). |
| `lib/validations/timeline.ts` | `TimelineUpdateSchema` (Zod schema) | Validates timeline update input: startDate and dueDate as ISO datetime strings. Refine rule: dueDate must be after startDate. |
| `lib/validations/plan.ts` | `planPhaseUpdateSchema` (Zod schema) | Validates phase update: title, description, objectives, resources — all non-empty strings. |
| `lib/validations/plan.ts` | `planTaskUpdateSchema` (Zod schema) | Validates task update: title, objective, instructions — all non-empty strings. |

---

### Which Functions Were Tested in Phase 1 & Phase 2?

| Function / Schema | Tested? | Where |
|---|---|---|
| `isDone` | **Indirectly only** | Never tested in isolation. It is called inside `calcPhaseProgress` and `calcOverallProgress` — its behaviour was observed through system-level tests but never unit-tested directly. |
| `calcPhaseProgress` | **Indirectly only** | Phase 2 TC-01 and TC-02 observed phase % updating via the UI. The function itself was never called in isolation with controlled inputs. |
| `calcOverallProgress` | **Indirectly only** | Phase 2 TC-01 observed overall % updating. Same issue — UI observation, not unit test. |
| `isPhaseUnlocked` | **Not tested** | TC-02 in Phase 2 checked the lock state in the UI, but the actual function was never tested. More critically: the function currently has its real logic commented out and `return true` hardcoded — this was not caught because no test asserts what the function *should* return. |
| `getPhaseStatus` | **Not tested** | Never written a test for this function despite it driving every status badge on every phase card. Analysed structurally in Step 2 but zero actual test cases exist. |
| `getDaysRemaining` | **Unit tested** | WB-01, WB-02, WB-03 in Phase 1 white box testing. The only function with real unit-level test cases. |
| `progressSchema` | **Indirectly only** | Decision table Row 3 (invalid status enum → 400) exercises this schema through the API, but the schema itself was never validated directly with edge inputs. |
| `TimelineUpdateSchema` | **Indirectly only** | BVA date field tests in Phase 2 exercise the refine rule through the `/admin/timeline` UI. The schema was never tested in isolation — e.g. what does it return for a non-datetime string? |
| `planPhaseUpdateSchema` | **Not tested** | Zero test coverage. No test case in Phase 1 or Phase 2 exercises phase update validation. |
| `planTaskUpdateSchema` | **Not tested** | Zero test coverage. No test case exercises task update validation. |

---

### Function Coverage Report

**Covered (unit-level):** 1 of 10 — `getDaysRemaining` only.

**Indirectly covered (system/integration-level, no isolation):** 5 of 10 — `isDone`, `calcPhaseProgress`, `calcOverallProgress`, `progressSchema`, `TimelineUpdateSchema`.

**Zero coverage:** 4 of 10 — `getPhaseStatus`, `isPhaseUnlocked`, `planPhaseUpdateSchema`, `planTaskUpdateSchema`.

```
  Function Coverage Summary
  ─────────────────────────────────────────────────────────
  getDaysRemaining          ████ Unit tested (WB-01/02/03)
  isDone                    ░░░░ Indirect only — no unit test
  calcPhaseProgress         ░░░░ Indirect only — no unit test
  calcOverallProgress       ░░░░ Indirect only — no unit test
  isPhaseUnlocked           ✗    Not tested — and currently broken
                                 (hardcoded return true, real logic commented out)
  getPhaseStatus            ✗    Not tested — 4 branches, 5 paths, zero coverage
  progressSchema            ░░░░ Indirect only — no schema unit test
  TimelineUpdateSchema      ░░░░ Indirect only — no schema unit test
  planPhaseUpdateSchema     ✗    Not tested at all
  planTaskUpdateSchema      ✗    Not tested at all
  ─────────────────────────────────────────────────────────
  ████ = Unit tested    ░░░░ = Indirect only    ✗ = Zero coverage
```

**The most critical gap: `isPhaseUnlocked`**

This function is commented out and hardcoded to `return true`. That means in the current codebase, every phase is accessible to every member regardless of their progress. If a test called `isPhaseUnlocked(2, phases, progressMap)` on a member who has not completed Phase 1, it would return `true` — and any test that did not *assert the expected lock state* would pass without catching the bug. This is exactly the scenario described in Step 6.

**Second most critical gap: `getPhaseStatus`**

Five distinct paths, drives every status label every member sees on every phase card, and has zero tests. A regression in any of the four `if` conditions — e.g. the `ON_TRACK` vs `BEHIND` calculation — would reach production undetected.

---

## Step 4 — Requirement Coverage

### All Requirements and User Stories from the Task Submission Feature Spec

Mapped against test cases from Phase 2 Assignment 2 (BVA, EP, Decision Table, State Transition).

---

#### User Stories

| # | User Story | Test Case(s) | Coverage |
|---|---|---|---|
| US-01 | As a member, I want to mark a task as complete so that my progress is saved and I can see how far I have come. | TC-01 (CHECKBOX toggle), ST-01, EP-01 | **Covered** |
| US-02 | As a member, I want to submit a written response for a task so that I can record what I learned. | TC-02 (TEXT submit/revise), ST-04, EP-02, BVA-T2 | **Covered** |
| US-03 | As a member, I want to submit a link as evidence so that I can attach a certificate, repo, or external resource. | TC-03 (LINK happy path), EP-03 | **Covered** |
| US-04 | As an admin, I want to see which tasks each member has submitted so that I can track team-wide progress. | No test case written for the admin member view. | **Not Covered** |

---

#### Business Rules

| # | Business Rule | Test Case(s) | Coverage |
|---|---|---|---|
| BR-01 | A member cannot submit an empty TEXT response. | TC-04 (empty TEXT blocked), EP-04, BVA-T1, DT Row 4 | **Covered** |
| BR-02 | A member cannot submit an invalid URL for a LINK task. | TC-03 (invalid URL — BUG-001), EP-06, EP-07, EP-08 | **Partially Covered** — valid blocking is tested. The known bug (EP-06) means the rule is violated in practice. The requirement is covered by tests but the implementation fails it. |
| BR-03 | A CHECKBOX task can be toggled on and off. | ST-01, ST-02, EP-01, TC-01 | **Covered** |
| BR-04 | A TEXT or LINK task can be revised after submission — new response replaces old. | TC-02 (revise flow), ST-05 | **Covered** |
| BR-05 | Progress percentage = tasks done ÷ total tasks × 100, rounded to nearest whole number. | TC-05 (progress accuracy across multiple submissions) | **Partially Covered** — tested via UI observation. The underlying `calcPhaseProgress` and `calcOverallProgress` functions have no unit tests asserting the rounding formula directly. |
| BR-06 | A task opened for the first time automatically moves from Not Started to In Progress. | ST-03 | **Covered** |

---

#### Error States

| # | Error State | Test Case(s) | Coverage |
|---|---|---|---|
| ES-01 | Submit clicked with empty TEXT field — button is disabled. | TC-04, EP-04, BVA-T1, DT Row 4 | **Covered** |
| ES-02 | Submit clicked with invalid URL — inline error shown, submission blocked. | TC-03 (invalid URL), EP-06, EP-07, EP-08 | **Partially Covered** — BUG-001 means the inline error is NOT shown for `https://,`. The test exists and is marked Expected Fail. |
| ES-03 | Submission fails due to network error — UI does not show success, input stays in field. | No test case written for network failure simulation. | **Not Covered** |
| ES-04 | Session expires mid-submission — member redirected to login, submission not saved. | No test case written for session expiry. | **Not Covered** |

---

#### Task Types — Feature Behaviour

| # | Requirement | Test Case(s) | Coverage |
|---|---|---|---|
| FT-01 | CHECKBOX: one click to complete, one click to undo. No text required. | ST-01, ST-02, TC-01, EP-01, EP-09 | **Covered** |
| FT-02 | TEXT: member types in textarea, clicks Submit. Response saved and visible on return. | TC-02, ST-04, EP-02, BVA-T2 through BVA-T5 | **Covered** |
| FT-03 | LINK: member submits a URL. Saved and visible on return. | TC-03, EP-03 | **Covered** |
| FT-04 | Both SUBMITTED and COMPLETED count as done for progress calculation. | TC-05, ST-01, ST-04 — both types contribute to % in these tests. | **Covered** |

---

#### Progress Display Requirements

| # | Requirement | Test Case(s) | Coverage |
|---|---|---|---|
| PD-01 | Phase page shows % of tasks completed in that phase. | TC-01, TC-02 (phase page check after submission) | **Covered** |
| PD-02 | Dashboard shows a progress bar per phase and an overall completion %. | TC-05, dashboard TC-01, TC-02 from Assignment 3 | **Covered** |
| PD-03 | Admin member view shows task-level status for each member. | No test case written. | **Not Covered** |
| PD-04 | Progress percentage updates without a page reload. | TC-05 explicitly checks this. | **Covered** |
| PD-05 | 100% of submitted tasks persist after page refresh. | TC-02 (refresh check), TC-03 (refresh check), EP-03 | **Covered** |

---

### Requirement Coverage Gap — Not Covered Requirements

These requirements have zero test cases:

| Requirement | Why It Matters |
|---|---|
| **US-04** — Admin view of member task submissions | The entire admin visibility use case is untested. An admin cannot currently verify their view is accurate, and no test would catch a regression in the admin data query. |
| **ES-03** — Network error — UI does not show success | This is a success metric in the spec ("zero cases where a failed submission shows a success state"). A network error is the most likely real-world scenario where a submission could silently fail. Untested. |
| **ES-04** — Session expiry mid-submission | Session handling during active form use is a common edge case. If a member writes a long TEXT response and their session expires, they lose their work and get redirected — but does the redirect happen cleanly? Untested. |
| **PD-03** — Admin member view shows task-level status | Same gap as US-04 from the admin perspective. The admin data pipeline from DB to view has no test coverage at any level. |

---

## Step 5 — Risk-Based Coverage

### Risk Matrix — All Major Features

| Feature | Risk Rating | Reasoning |
|---|---|---|
| **Task Submission (TEXT, LINK, CHECKBOX)** | 🔴 High | This is the core action the entire app exists to support. A bug here — silent save failure, wrong state written, progress not updating — directly breaks a member's ability to record their work. Data loss risk is real. |
| **Progress Tracking** (`calcPhaseProgress`, `calcOverallProgress`) | 🔴 High | Every number a member and admin sees — phase %, overall %, task counts — comes from these two functions. A rounding bug or wrong `isDone` logic silently gives every user wrong data. There is no UI signal that the calculation is wrong. |
| **Login / Authentication** | 🔴 High | If login breaks, no member can access the platform at all. If auth is bypassed, any user can access any data. Total access failure or data exposure — both are catastrophic. |
| **Phase Navigation & Lock State** (`isPhaseUnlocked`) | 🔴 High | Controls which phases a member can access. Currently `return true` is hardcoded — the real lock logic is commented out. If this ships to production without being restored, all phases are open to all members regardless of progress, breaking the intended curriculum structure. |
| **Dashboard** | 🟡 Medium | Primarily a display layer — reads and renders data calculated elsewhere. A bug here is visible immediately (wrong number shown) but does not corrupt stored data. Still important but failure is observable and recoverable. |
| **Admin View** | 🟡 Medium | Only affects the admin role — one or two users. A bug here does not affect member data or progress. It is a visibility/reporting failure, not a data integrity failure. Important to fix but not a blocker for members. |
| **Phase Page (task list view)** | 🟡 Medium | Renders the task list for a phase. A bug here might show tasks in the wrong order or fail to render a task, but member progress data is not at risk — it is a display issue. |
| **Timeline / Status Labels** (`getPhaseStatus`) | 🟡 Medium | Drives ON_TRACK, BEHIND, OVERDUE labels on phase cards. Wrong labels are confusing and erode trust, but they do not corrupt data or block access. Still — the function has 5 distinct paths and zero tests, which makes medium-risk feel underweighted in practice. |
| **Zod Validation Schemas** (`progressSchema`, `TimelineUpdateSchema`) | 🟡 Medium | Acts as the last line of defence before bad data reaches the DB. If `progressSchema` lets an invalid status enum through, or `TimelineUpdateSchema` fails to enforce dueDate > startDate, corrupted data enters the system. Schemas are simple but their failure has High-risk downstream consequences. |
| **Pinned Resource Link / Static UI** | 🟢 Low | A broken link is annoying but has zero impact on data, progress, or access. Any member can navigate directly. Lowest possible failure consequence. |

---

### Are High-Risk Features Covered the Most?

**Honest answer: No.**

| Feature | Risk | Coverage Level |
|---|---|---|
| Task Submission | 🔴 High | System-level only. Core validation function (URL check) has a known bug. No unit tests on submission logic. |
| Progress Tracking | 🔴 High | Zero unit tests. `calcPhaseProgress` and `calcOverallProgress` are only observed indirectly through the UI. |
| Login / Auth | 🔴 High | 5 system-level tests + 2 integration-level tests from Phase 1. Best-covered High-risk feature, but still no unit tests on auth helpers. |
| Phase Lock State | 🔴 High | 1 UI-level check (TC-02 Phase 2). The function itself (`isPhaseUnlocked`) is hardcoded to `return true` and has no test asserting it should lock anything. |
| Dashboard | 🟡 Medium | 6 system-level tests from Assignment 3. Over-covered relative to its risk level. |
| Admin View | 🟡 Medium | Zero coverage — not a single test case for the admin data view. |
| Timeline Status | 🟡 Medium | Zero unit tests. The function was analysed structurally in Step 2 but never tested. |
| Zod Schemas | 🟡 Medium | Indirectly tested through system/integration flows only. No direct schema unit tests. |

**The gap in plain terms:**

The test suite is heaviest on the dashboard (Medium risk, 6 tests) and login (High risk, 7 tests). The features that most directly handle data integrity — progress calculation, task submission validation, and phase lock logic — are either indirectly covered or not covered at all. The two functions that would catch the most critical failures in production (`calcPhaseProgress` and `isPhaseUnlocked`) have zero unit tests between them.

This is the natural result of an inverted pyramid: system-level tests through the UI are easy to write and give you confidence in visible behaviour, but they do not reach the logic that matters most until it is already wrong in production.

---

## Step 6 — Coverage vs Confidence

### Kristi's Bug: Phase 2 Was Accessible Without Completing Phase 1

The bug: a member could navigate to Phase 2 (and beyond) without completing Phase 1. The intended behaviour — each phase is locked until the previous one is 100% complete — was not enforced.

Looking at the current code, this bug is not hypothetical. `isPhaseUnlocked` in `lib/utils/progress.ts` currently reads:

```ts
export function isPhaseUnlocked(
  phaseOrder: number,
  phases: { order: number; tasks: { id: number }[] }[],
  progressMap: ProgressMap
): boolean {
  return true // for testing, unlock all phases
  // if (phaseOrder === 1) return true
  // const prev = phases.find((p) => p.order === phaseOrder - 1)
  // if (!prev) return true
  // return prev.tasks.every((t) => isDone(t.id, progressMap))
}
```

The real lock logic is commented out. The function always returns `true`.

---

### Would a Passing Test Have Caught This Bug?

Imagine this test exists and passes:

```ts
it('calls isPhaseUnlocked for phase 2', () => {
  const result = isPhaseUnlocked(2, phases, progressMap)
  // no assertion on the result
  expect(result).toBeDefined()
})
```

**This test passes. The bug survives.**

`isDefined()` is true — `true` is defined. The function ran. Every line inside it was executed. Line coverage: 100%. Function coverage: 100%. The test passes cleanly and the bug is invisible.

---

### Why Line or Function Coverage Alone Would Not Catch This

**Function coverage** only tells you whether the function was called at all. It says nothing about whether what it returned was correct. `isPhaseUnlocked(2, phases, emptyProgressMap)` returns `true` — function was called, function returned a value, coverage metric is satisfied. The fact that it should have returned `false` is not part of the coverage metric.

**Line coverage** only tells you whether each line of code was executed. In `isPhaseUnlocked`, there is only one executable line: `return true`. A test that calls the function hits that line. 100% line coverage. The commented-out lines are not executable and do not factor into the metric. The coverage tool reports a green tick. The bug is still there.

The core problem: **coverage metrics measure execution, not correctness.** A function that always returns `true` achieves 100% line and function coverage with a single call. The coverage metric cannot tell the difference between a function that returns the right thing and a function that always returns the same wrong thing.

---

### What Specific Assertion Would Actually Catch This Bug

The test needs to verify the *return value* for a specific real-world scenario where the answer must be `false`:

```ts
it('locks phase 2 when phase 1 is not complete', () => {
  const phases = [
    { order: 1, tasks: [{ id: 1 }, { id: 2 }] },
    { order: 2, tasks: [{ id: 3 }] },
  ]
  const progressMap = {} // member has completed nothing

  const result = isPhaseUnlocked(2, phases, progressMap)

  expect(result).toBe(false) // ← this is the assertion that catches the bug
})
```

And the complementary test that confirms the unlock condition works when it should:

```ts
it('unlocks phase 2 when phase 1 is fully complete', () => {
  const phases = [
    { order: 1, tasks: [{ id: 1 }, { id: 2 }] },
    { order: 2, tasks: [{ id: 3 }] },
  ]
  const progressMap = {
    1: { status: 'COMPLETED' },
    2: { status: 'SUBMITTED' },
  }

  const result = isPhaseUnlocked(2, phases, progressMap)

  expect(result).toBe(true) // unlocked because phase 1 is done
})
```

With the current hardcoded `return true`, **the first test fails immediately** — `expect(true).toBe(false)` — and the bug is surfaced before it ever ships.

---

### The Difference Between a Test That Runs Code and a Test That Proves Something

**A test that runs code** calls a function, confirms it does not throw an error, and checks that something was returned. It satisfies coverage metrics. It gives you a green tick. It tells you the code path is reachable.

**A test that proves something** asserts a specific output for a specific, meaningful input — and it is written for inputs where the function *could plausibly return the wrong thing*. It is designed to fail if the logic is wrong.

The difference is in the assertion. `expect(result).toBeDefined()` is a test that runs code. `expect(result).toBe(false)` for a member who has not completed Phase 1 is a test that proves something.

Kristi's bug survives 100% line coverage, 100% function coverage, and even 100% branch coverage if the only branch executed is `return true`. It would only be caught by a test that:

1. Sets up a state where the correct answer is unambiguously `false`
2. Asserts that the function returns `false` for that state
3. Would *fail* if the function returned `true`

The bug is not in the code coverage gap. The bug is in the assertion gap. Coverage tells you which code ran. Assertions tell you whether the code was right. You can have 100% of the first and 0% of the second — and every bug Kristi found would survive that test suite untouched.

---

*Submitted by Deepti · Day 5 Assignment 1 · All Steps · June 2026*
