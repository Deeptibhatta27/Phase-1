# Phase 3 Day 3 — Assignment 1: Read Three Functions
**BetaninjasSEOLMS · lib/utils/timeline.ts · lib/utils/progress.ts · lib/validations/progress.ts · Deepti · June 2026**

---

## Function 1 — `getPhaseStatus` in `lib/utils/timeline.ts`

```ts
export function getPhaseStatus(
  progressPercent: number,
  startDate: Date,
  dueDate: Date,
  now: Date = new Date()
): TimelineStatus
```

---

### Inputs

| Parameter | Type | Default Value | Notes |
|---|---|---|---|
| `progressPercent` | `number` | None — required | Expected range 0–100. Represents % of tasks done in the phase. |
| `startDate` | `Date` | None — required | The date the phase officially begins. |
| `dueDate` | `Date` | None — required | The deadline for the phase. |
| `now` | `Date` | `new Date()` | **This one has a default value.** If not passed, it uses the current system time. Passing it explicitly is what makes this function testable — without it you could not control "what time is it now" in a test. |

---

### Outputs

The return type is `TimelineStatus`, which is a TypeScript union type defined as:

```ts
type TimelineStatus = 'NOT_STARTED' | 'ON_TRACK' | 'BEHIND' | 'COMPLETED' | 'OVERDUE'
```

All 5 possible return values are string literals. The function always returns exactly one of these — it cannot return `undefined`, `null`, or any other value.

| Return Value | Meaning |
|---|---|
| `'COMPLETED'` | The phase is 100% done |
| `'NOT_STARTED'` | Current time is before the phase start date |
| `'OVERDUE'` | Deadline has passed and phase is not 100% done |
| `'ON_TRACK'` | Phase is in progress and progress % is keeping up with time elapsed |
| `'BEHIND'` | Phase is in progress but progress % is lagging behind time elapsed |

---

### Branches — Every Path Written as a Sentence

Reading the function line by line:

```ts
if (progressPercent === 100) return 'COMPLETED'
if (now < startDate) return 'NOT_STARTED'
if (now > dueDate && progressPercent < 100) return 'OVERDUE'
// ... calculation ...
return progressPercent >= timeElapsedPercent ? 'ON_TRACK' : 'BEHIND'
```

**Path 1:** If `progressPercent` equals exactly 100, the function immediately returns `'COMPLETED'` without checking any dates.

**Path 2:** If `now` is before `startDate` (the phase has not begun yet), the function returns `'NOT_STARTED'`. This only fires if Path 1 did not — meaning progress is not 100%.

**Path 3:** If `now` is after `dueDate` AND `progressPercent` is less than 100, the function returns `'OVERDUE'`. Both conditions must be true simultaneously.

**Path 4:** If none of the above conditions are true — meaning the phase is in progress, the deadline has not passed, and progress is not 100% — the function calculates how much time has elapsed as a percentage of total duration, then returns `'ON_TRACK'` if `progressPercent >= timeElapsedPercent`.

**Path 5:** Same entry condition as Path 4, but `progressPercent < timeElapsedPercent` — the member is not keeping up with the pace the timeline requires — so the function returns `'BEHIND'`.

**Flow diagram:**

```
  start
    │
    ├─ progressPercent === 100 ──────────────────────────→ 'COMPLETED'
    │
    ├─ now < startDate ──────────────────────────────────→ 'NOT_STARTED'
    │
    ├─ now > dueDate && progressPercent < 100 ───────────→ 'OVERDUE'
    │
    └─ (none of above — phase is live and on-going)
         │
         ├─ progressPercent >= timeElapsedPercent ────────→ 'ON_TRACK'
         └─ progressPercent < timeElapsedPercent  ────────→ 'BEHIND'
```

---

### Minimum Tests to Cover Every Path

**Answer: 5 test cases.**

One test per path. Each path leads to a different return value, and each return value requires a distinct combination of inputs. There is no way to reach `'BEHIND'` with the same inputs that reach `'ON_TRACK'`, and there is no way to reach `'COMPLETED'` without `progressPercent === 100`. The five paths are mutually exclusive and together exhaustive — every possible return value maps to exactly one path.

| Test | Inputs | Expected Return |
|---|---|---|
| T1 | `progressPercent = 100`, any dates | `'COMPLETED'` |
| T2 | `progressPercent = 0`, `now = before startDate` | `'NOT_STARTED'` |
| T3 | `progressPercent = 50`, `now = after dueDate` | `'OVERDUE'` |
| T4 | `progressPercent = 60`, `now = 50% of the way through the duration` | `'ON_TRACK'` (progress ahead of time) |
| T5 | `progressPercent = 30`, `now = 50% of the way through the duration` | `'BEHIND'` (progress behind time) |

5 tests = 100% path coverage = 100% branch coverage = 100% line coverage for this function.

---

### Data Type Edge Case: What If `progressPercent` Is the String `"100"`?

If `progressPercent` is passed as the string `"100"` instead of the number `100`, **the first branch does not fire.**

The condition is:
```ts
if (progressPercent === 100)
```

This uses **strict equality** (`===`), which checks both value and type in JavaScript/TypeScript. `"100" === 100` is `false` — a string is never strictly equal to a number. So the function falls through to the second branch.

What happens next depends on the remaining checks:
- `now < startDate` — this comparison works fine with Date objects regardless of `progressPercent`
- `now > dueDate && progressPercent < 100` — `"100" < 100` evaluates to `false` in JavaScript because `"100"` coerces to `100` for the `<` operator, which makes `100 < 100` false. So this branch also does not fire.
- The function reaches the final ternary: `"100" >= timeElapsedPercent`. JavaScript coerces `"100"` to `100` for `>=`, so this comparison works — but the function never returns `'COMPLETED'` for a phase that is actually 100% done.

**The consequence:** a phase that is fully complete would be shown as `'ON_TRACK'` or `'BEHIND'` instead of `'COMPLETED'`. The celebration banner on the dashboard would never appear. Progress would appear incorrect.

**In TypeScript** this specific mistake would be caught at compile time if `progressPercent` is typed as `number` — the type system would reject passing a string. But if the value comes from an API response that is not validated (e.g. JSON where all values are strings), TypeScript's compile-time protection does not apply at runtime. This is exactly why Zod schemas exist at the API boundary.

---

## Function 2 — `calcPhaseProgress` in `lib/utils/progress.ts`

```ts
export function calcPhaseProgress(
  tasks: { id: number }[],
  progressMap: ProgressMap
): number
```

---

### Inputs

**`tasks`:** An array of objects where each object has at least one property: `id` of type `number`. The function only needs the `id` from each task — it does not care about task type, title, or any other property. An empty array `[]` is a valid input.

**`progressMap`:** A `Record<number, { status: string }>` — a plain JavaScript object where the keys are task IDs (numbers) and the values are objects with a `status` string property. It represents the current state of each task for a specific member. A task ID that does not appear in the map means that task has no recorded progress — `progressMap[taskId]` returns `undefined`.

---

### Output

Returns a `number` — a whole integer representing the completion percentage of the phase.

**Possible range:** 0 to 100, inclusive. Always an integer because `Math.round` is applied. It cannot return a negative number or a number above 100 given valid inputs.

- Minimum: `0` — no tasks done, or the zero-task branch
- Maximum: `100` — all tasks done
- Example intermediate: `33` for 1 of 3 tasks done (Math.round(0.333... × 100) = 33)

---

### The Branch — Explained

```ts
if (tasks.length === 0) return 0
```

**What it checks:** Whether the tasks array is empty.

**What it returns:** `0` immediately, without doing any calculation.

**Why this branch exists:** Without it, the next line would execute `done / tasks.length`, which is `0 / 0` — a division by zero. In JavaScript, `0 / 0` returns `NaN` (Not a Number), not an error. `Math.round(NaN)` also returns `NaN`. So without this guard, a phase with no tasks would return `NaN` — and `NaN` displayed as a progress percentage would break every UI component that tries to render or compare it. The branch is a defensive guard that prevents a mathematically undefined operation from silently producing garbage output.

---

### `isDone` — Explained

```ts
function isDone(taskId: number, progressMap: ProgressMap): boolean {
  const status = progressMap[taskId]?.status
  return status === "SUBMITTED" || status === "COMPLETED"
}
```

**The two status values treated as done:** `"SUBMITTED"` and `"COMPLETED"`.

**What this means for the question "can a task be done without being COMPLETED?"**

Yes. A TEXT or LINK task that has been submitted is in the `SUBMITTED` state — not `COMPLETED`. `COMPLETED` is exclusively for CHECKBOX tasks. Yet `isDone` returns `true` for both. This means a member who submits a written response has their task counted as done for progress calculation, even though the status string is `SUBMITTED` not `COMPLETED`. The two states are functionally equivalent from a progress standpoint — both count toward the percentage — but they are semantically different and map to different task types.

This also means `isDone` returns `false` for `NOT_STARTED`, `IN_PROGRESS`, any undefined task ID, and any unexpected status string. If the status were `"done"` (lowercase) or `"complete"` (no D), `isDone` would return `false` — a silent failure. The strict string equality is both reliable and brittle.

---

### Edge Case: Every Task Is SUBMITTED, None Are COMPLETED

`calcPhaseProgress` does not distinguish between `SUBMITTED` and `COMPLETED`. It calls `isDone` for each task, and `isDone` returns `true` for both. If a phase has 4 tasks and all 4 are in the `SUBMITTED` state, `done` equals 4, `tasks.length` equals 4, and the calculation is `Math.round((4 / 4) * 100)` = `100`.

**The function returns `100`.** A phase where every task is `SUBMITTED` and zero are `COMPLETED` shows 100% progress. This is correct behaviour per the spec: "For progress calculation — both Submitted and Completed count as done."

---

### Isolation Question — Real Database or Fake?

**You would never use the real database in a unit test for `calcPhaseProgress`. You pass in a plain JavaScript object as `progressMap` instead.**

The reason is fundamental to what a unit test is for. A unit test is supposed to test one function's logic in isolation — its inputs, its calculation, its output. The moment you involve a real database, you are no longer testing `calcPhaseProgress`; you are testing `calcPhaseProgress` + the DB connection + the query + the ORM + the network. If the test fails, you do not know which part broke. If the database is unavailable, the test fails — not because the function is wrong, but because the environment is wrong. Unit tests must be deterministic: same inputs, same outputs, every time, with no external dependencies.

What you pass in instead is a hand-crafted `progressMap` object that represents exactly the state you want to test:

```ts
// Test: 2 of 3 tasks done
const tasks = [{ id: 1 }, { id: 2 }, { id: 3 }]
const progressMap = {
  1: { status: "SUBMITTED" },
  2: { status: "COMPLETED" },
  // task 3 not in map — means NOT_STARTED
}
const result = calcPhaseProgress(tasks, progressMap)
expect(result).toBe(67)
```

This is called a **test double** or **mock** — not a mock of a library, but a plain fake data structure that replaces the real data source. `ProgressMap` is just `Record<number, { status: string }>` — a plain object. You do not need to mock anything. You just construct the object you want and pass it in. This is one of the cleanest things about how this function is designed: it takes data as a parameter instead of fetching it internally, which makes it naturally testable without any test framework setup beyond the assertion itself.

---

## Function 3 — `isPhaseUnlocked` in `lib/utils/progress.ts`

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

---

### What It Currently Does

Every single time this function is called — regardless of `phaseOrder`, regardless of what is in `phases`, regardless of what is in `progressMap` — it returns `true`. The very first line is `return true`, which means the function exits immediately. The three parameters are accepted but never read. The commented-out logic below the `return true` line is unreachable code; it does not execute. No phase is ever locked. Every member has access to every phase at all times.

---

### What It Is Supposed to Do

Reading the commented-out logic:

```ts
// if (phaseOrder === 1) return true
// const prev = phases.find((p) => p.order === phaseOrder - 1)
// if (!prev) return true
// return prev.tasks.every((t) => isDone(t.id, progressMap))
```

**Line by line:**

1. **Phase 1 is always unlocked** — the first phase is the entry point, there is nothing to complete before it, so it is unconditionally accessible.
2. **Find the previous phase** — for any phase N, look up phase N-1 in the phases array.
3. **If the previous phase does not exist** — return `true` as a safe fallback (do not block access if data is missing).
4. **Check that every task in the previous phase is done** — only unlock the current phase if every single task in the preceding phase has been submitted or completed.

The intended behaviour: phase progression is sequential and earned. A member must finish every task in Phase 1 before Phase 2 unlocks, finish every task in Phase 2 before Phase 3 unlocks, and so on. Partial completion does not unlock the next phase — it must be 100%.

---

### Why This Is a Bug — Connection to Kristi's Phase 1 Finding

Kristi's finding was: Phase 2 was accessible without completing Phase 1. That is exactly what this bug produces. With `return true` hardcoded, `isPhaseUnlocked(2, phases, progressMap)` returns `true` for a member who has completed zero tasks. The phase cards on the dashboard show all phases as unlocked. A member can click into Phase 5 on day one.

This is not a subtle bug. It is a complete bypass of the curriculum's core structure. The learning path is designed to be sequential — each phase builds on the previous one. Skipping ahead means a member is attempting advanced topics without the foundation, which defeats the purpose of the programme entirely.

The comment `// for testing, unlock all phases` suggests this was an intentional temporary change made during development so the developer could navigate freely without completing tasks. It was never reverted before the code shipped. This is one of the most common sources of production bugs: a development shortcut left in place.

---

### What a Unit Test Would Need to Prove

If the real logic were uncommented, this specific test case would catch the bug and verify the fix:

```ts
it('locks phase 2 when phase 1 is not complete', () => {
  const phases = [
    { order: 1, tasks: [{ id: 1 }, { id: 2 }] },
    { order: 2, tasks: [{ id: 3 }] },
  ]
  const progressMap = {} // member has done nothing

  const result = isPhaseUnlocked(2, phases, progressMap)

  expect(result).toBe(false)
})
```

**With the current hardcoded `return true`:** this test fails — `expect(true).toBe(false)`. The failure message immediately surfaces the bug.

**With the real logic uncommented:** `prev.tasks.every((t) => isDone(t.id, progressMap))` evaluates `isDone(1, {})` and `isDone(2, {})` — both return `false` because neither task ID exists in the empty map. `every` returns `false`. The function returns `false`. The test passes.

The complementary test that confirms the unlock works correctly:

```ts
it('unlocks phase 2 when all phase 1 tasks are done', () => {
  const phases = [
    { order: 1, tasks: [{ id: 1 }, { id: 2 }] },
    { order: 2, tasks: [{ id: 3 }] },
  ]
  const progressMap = {
    1: { status: 'COMPLETED' },
    2: { status: 'SUBMITTED' },
  }

  const result = isPhaseUnlocked(2, phases, progressMap)

  expect(result).toBe(true)
})
```

Both tests together prove the lock logic works in both directions. Neither test touches a database. Neither test needs a running server. Both pass or fail purely on the function's logic.

---

## Data Types — Findings from `lib/validations/progress.ts`

```ts
export const progressSchema = z.object({
  taskId: z.number().int().positive(),
  status: z.enum(["NOT_STARTED", "IN_PROGRESS", "SUBMITTED", "COMPLETED"]),
  response: z.string().optional(),
  link: z.string().url().optional(),
})
```

---

### Question 1: What happens if `taskId` is sent as the string `"5"` instead of the number `5`?

**Zod rejects it.** `z.number()` does not coerce by default. If the API receives `{ taskId: "5" }`, Zod's `progressSchema.parse()` throws a `ZodError` with a message like: `"Expected number, received string"` at the `taskId` path.

This matters at API boundaries. JSON payloads from forms or query strings often arrive with numeric-looking values as strings. If the submission form sends `taskId` as a string (which can happen if it is read from a URL parameter or a form field's `.value` property without explicit conversion), the schema will reject the request with a 400 before it reaches the database. This is the correct behaviour — but the client must ensure it sends a number, not a string.

If Zod coercion is needed, the schema would need `z.coerce.number().int().positive()` — but that is a deliberate design decision, not the default.

---

### Question 2: What happens if `taskId` is `-1`?

**Zod rejects it.** The schema is `z.number().int().positive()`. The `.positive()` validator requires the number to be strictly greater than zero. `-1` fails this check. Zod throws a `ZodError`: `"Number must be greater than 0"`.

`0` would also be rejected by `.positive()` — Zod's `.positive()` means `> 0`, not `>= 0`. If zero were a valid task ID, the schema would need `.nonnegative()` instead.

---

### Question 3: What happens if `status` is sent as `"DONE"` instead of `"SUBMITTED"`?

**Zod rejects it.** `z.enum(["NOT_STARTED", "IN_PROGRESS", "SUBMITTED", "COMPLETED"])` is a closed set — only these exact four string values are valid. `"DONE"` is not in the enum. Zod throws a `ZodError`: `"Invalid enum value. Expected 'NOT_STARTED' | 'IN_PROGRESS' | 'SUBMITTED' | 'COMPLETED', received 'DONE'"`.

This is exactly the protection that should have prevented BUG-001 at the server level — any string that is not a valid status enum is rejected before reaching the database. The risk is on the client side: if the front-end sends `"DONE"` by mistake (a typo, a legacy value, a copy-paste from another system), the API correctly rejects it.

---

### New Decision Table Entry — Data Type Edge Cases

The Phase 2 decision table covered 4 conditions: authenticated, valid task ID, valid status enum, non-empty text. Based on the schema analysis, a fifth condition should be added:

**Condition: `taskId` is the correct data type (number, not string)**

| Authenticated | Task ID Valid | Task ID Correct Type | Status Valid | Text Non-Empty | Expected Outcome |
|---|---|---|---|---|---|
| YES | YES | **NO — string `"5"`** | YES | YES | **400 Bad Request** — Zod rejects `taskId: "5"`. Error: `"Expected number, received string"`. Submission blocked before DB is touched. |

**Why this matters:** In a web form, task IDs often come from URL parameters or hidden inputs — both of which are strings in HTML. Without explicit `parseInt()` or `Number()` conversion on the client, `taskId` arrives as a string. The schema catches it, but the developer must know to convert on the client side. This is a real integration gap between front-end and API that should be tested explicitly — not assumed to work because the type system says so.

---

*Submitted by Deepti · Phase 3 Day 3 · Assignment 1 · June 2026*
