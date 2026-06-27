# Phase 3 Day 3 — Assignment 3: Find What Was Never Handled
**BetaninjasSEOLMS · TextSubmission.tsx · LinkSubmission.tsx · route.ts · Deepti · June 2026**
*(Step 3 — PR #28 to be completed separately)*

---

## Step 1 — The Silent Error

### `TextSubmission.tsx` — `handleSubmit` try/catch analysis

```ts
async function handleSubmit(e: React.FormEvent) {
  e.preventDefault()
  if (!response.trim()) return
  setLoading(true)
  setSaved(false)
  try {
    const fullResponse = existingNote
      ? `${response.trim()}\n[NOTE]: ${existingNote}`
      : response.trim()
    const res = await fetch("/api/progress", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ taskId, status: "SUBMITTED", response: fullResponse }),
    })
    if (res.ok) {
      setSaved(true)
      router.refresh()
    }
  } finally {
    setLoading(false)
  }
}
```

---

**What is inside the try block — what is being attempted:**

Two things:
1. Building the `fullResponse` string — if `existingNote` is present, it appends it to the member's typed response with a `[NOTE]:` separator. Otherwise it just trims the response.
2. Making a `fetch` POST request to `/api/progress` with the task ID, status `"SUBMITTED"`, and the response text. If the server responds with `res.ok` (status 200–299), it sets `saved` to `true` and calls `router.refresh()` to reload the page data.

---

**What happens in the catch block — what does the code do when something goes wrong:**

**There is no catch block.** The function uses `try { ... } finally { ... }` — a try/finally with no catch. This means any error thrown inside the try block — a network failure, a `fetch` rejection, a JSON serialisation error — is not caught. It propagates up the call stack as an unhandled promise rejection. The browser logs it to the console. The member sees nothing.

---

**What the finally block does:**

```ts
finally {
  setLoading(false)
}
```

It sets `loading` back to `false`, which re-enables the Submit button and removes the spinner. This runs regardless of whether the fetch succeeded or failed. Its purpose is to ensure the button is never permanently stuck in a loading/disabled state.

---

**The gap — what the member sees when submission fails:**

Almost nothing useful. Here is what actually happens on failure:

| Failure scenario | What the member sees |
|---|---|
| Network offline — `fetch` throws `TypeError: Failed to fetch` | The spinner disappears (finally runs). The button re-enables. No error message. `saved` remains `false` so "Submitted!" never appears. The member stares at a form that looks exactly like it did before they clicked Submit — with no indication of what happened. |
| Server returns 500 | `res.ok` is false, so `setSaved(true)` is never called. But there is no `else` branch — no error state is set. Same result: spinner disappears, button re-enables, silence. |
| Server returns 401 (session expired) | Same — `res.ok` is false, nothing happens. The member is not redirected to login. They do not know their session expired. |
| Server returns 400 (validation failure) | Same — `res.ok` is false, nothing visible happens. |

The spec says: *"UI does not show success — member sees their input still in the field."* The input does stay in the field (state is not cleared), so the spec's minimum requirement is technically met. But showing no error at all is not the same as handling the error. A member who clicked Submit and sees nothing has no idea whether they should try again, whether their work was lost, or whether the app is broken.

---

**What good error handling would look like:**

```ts
const [error, setError] = useState<string | null>(null)

async function handleSubmit(e: React.FormEvent) {
  e.preventDefault()
  if (!response.trim()) return
  setLoading(true)
  setSaved(false)
  setError(null)
  try {
    const res = await fetch("/api/progress", { ... })
    if (res.ok) {
      setSaved(true)
      router.refresh()
    } else if (res.status === 401) {
      setError("Your session has expired. Please refresh the page and log in again.")
    } else {
      setError("Something went wrong. Your response is still here — please try again.")
    }
  } catch {
    setError("Could not reach the server. Check your connection and try again.")
  } finally {
    setLoading(false)
  }
}
```

The member should see a specific, human-readable message below the submit button. Their typed text must remain in the field. The error message must distinguish between a network failure and a server error — these require different actions from the member. The form must never look identical after a failure as it does before a submission attempt.

---

**`LinkSubmission.tsx` — same pattern?**

`LinkSubmission.tsx` has the exact same `try { ... } finally { setLoading(false) }` structure with no catch block — the error handling is identical: silent on failure, spinner disappears, button re-enables, no error message shown to the member.

---

## Step 2 — The API Gap

### `route.ts` POST handler — database error analysis

```ts
const progress = await prisma.progress.upsert({
  where: { userId_taskId: { userId: session.user.id, taskId } },
  update: { ... },
  create: { ... },
})

return NextResponse.json(progress)
```

---

**Is there a try/catch around the database call?**

**No.** The `prisma.progress.upsert()` call has no try/catch, no error boundary, and no fallback. It is a bare `await` inside the POST handler with nothing protecting it.

---

**What happens if the database is unavailable and this call throws:**

Prisma throws a `PrismaClientInitializationError` or `PrismaClientKnownRequestError`. Because there is no catch block, the error propagates up as an unhandled promise rejection inside the Next.js API route. Next.js catches unhandled errors in API routes and returns a `500 Internal Server Error` response. The response body is Next.js's default error JSON — something like `{ "message": "Internal Server Error" }` — with no detail about what failed.

Other scenarios that also throw with no handling:
- `taskId` does not exist in the `Task` table → Prisma foreign key constraint error → 500
- Database connection pool exhausted → 500
- Unique constraint race condition (two simultaneous requests for the same userId+taskId during the brief window before upsert resolves) → 500

---

**What the member experiences:**

The `fetch` in `TextSubmission.tsx` receives a 500 response. `res.ok` is `false` (500 is not in the 200–299 range). The component's `if (res.ok)` branch does not execute. `setSaved(true)` is never called. Because there is no catch and no error state in the component, the member sees the spinner disappear and the button re-enable — exactly as described in Step 1. The member has no idea whether their submission reached the server, failed at the DB, or never left their browser.

The combination of the API throwing an unhandled 500 and the component having no error handling means the failure is completely invisible end-to-end.

---

**What correct error handling should look like in the API:**

```ts
export async function POST(request: Request) {
  const session = await auth()
  if (!session?.user?.id) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  }

  let body: unknown
  try {
    body = await request.json()
  } catch {
    return NextResponse.json({ error: "Invalid JSON in request body" }, { status: 400 })
  }

  const parsed = progressSchema.safeParse(body)
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 })
  }

  const { taskId, status, response, link } = parsed.data

  try {
    const progress = await prisma.progress.upsert({
      where: { userId_taskId: { userId: session.user.id, taskId } },
      update: { ... },
      create: { ... },
    })
    return NextResponse.json(progress)
  } catch (err) {
    console.error("[POST /api/progress] DB error:", err)
    return NextResponse.json(
      { error: "Failed to save progress. Please try again." },
      { status: 500 }
    )
  }
}
```

The server should:
1. Catch the DB error and log it server-side for debugging
2. Return a structured JSON error response with status 500 — not Next.js's default unhandled error
3. Never expose internal Prisma error details to the client

The member should: see the error message surfaced by the component (once the component's error handling is also fixed) — something like "Something went wrong. Your response is still here — please try again."

---

## Step 4 — The Question

### Which gap is worse for the member?

**The unhandled API crash (Step 2) is worse — but the silent UI error (Step 1) is what makes it invisible.**

They are two halves of the same failure. Here is why the API gap is the more serious of the two:

The component's missing catch block means the member sees no error message — but the member's typed text is still in the field. They can try again. The failure is annoying but recoverable from the member's perspective, as long as the server eventually responds.

The API's missing try/catch means that when the database fails, the server returns an unhandled 500 with no structured error body. This has two consequences beyond the member's experience: it means the failure is unlogged (or logged generically by Next.js without context), so the engineering team has no visibility into why submissions are failing or how often. A member retrying after a 500 may retry successfully once the DB recovers — or they may retry into the same failure repeatedly, with no way to know which is happening.

The deeper reason the API gap is worse: **it affects data integrity, not just UI feedback.** A DB failure means the progress row was never written. The member's task remains `IN_PROGRESS` in the database while the UI reset to its pre-submission state. If the member does not notice and navigates away, their progress is silently not saved. The spec success metric — *"100% of submitted tasks persist after page refresh"* — is violated without any visible signal.

---

### Which gap is easier to fix?

**The UI component fix is easier** — it is a self-contained React state change. Adding an `error` state variable, a catch block, and a conditional error message below the button is approximately 10 lines of code per component. No schema changes, no database migrations, no API contract changes. Both `TextSubmission.tsx` and `LinkSubmission.tsx` can be fixed in the same PR with a shared error handling pattern.

The API fix is slightly more work because it requires deciding on an error logging strategy (what to log, where), handling multiple error types from Prisma distinctly if needed, and ensuring the error response shape is consistent with how the components read it. Still straightforward — but it touches the server layer and requires a decision about error observability, not just a UI state addition.

---

### One test case that catches both gaps simultaneously

**TC: TEXT task submission fails due to server error — member sees an error message and data is not lost**

```
Precondition:
  Member is logged in. A TEXT task is open and in IN_PROGRESS state.
  The database is made unavailable (simulated by: intercepting the POST
  request with a mock that returns HTTP 500, OR by taking the DB offline
  in a test environment).

Steps:
  1. Navigate to a TEXT task page.
  2. Type a response: "This is my test submission."
  3. Click "Submit Response".
  4. Observe the UI immediately after the click.
  5. Wait for the loading spinner to disappear.
  6. Observe the UI state after the spinner clears.
  7. Observe the text in the textarea.
  8. Reload the page.
  9. Observe the task status after reload.

Expected — what this test proves:
  Step 4: Loading spinner appears and Submit button is disabled. (Correct.)
  Step 6: An error message is visible — something like "Something went wrong.
           Your response is still here — please try again." The "Submitted!"
           green confirmation does NOT appear.
  Step 7: The textarea still contains "This is my test submission." — the
           member's text has not been cleared.
  Step 8–9: After reload, the task status is still IN_PROGRESS — the failed
             submission did not write to the database.

What this test catches:
  - Component gap (Step 1): if no error message appears at Step 6, the
    silent UI failure is confirmed. The test fails.
  - API gap (Step 2): if the task shows SUBMITTED after reload at Step 9,
    the DB write somehow succeeded despite the 500 — inconsistent state.
    If it shows IN_PROGRESS but "Submitted!" appeared at Step 6, the
    component showed a false success on a failed request. Either way the test fails.
  - Both gaps together: the test only passes when the component correctly
    surfaces the server error AND the database correctly did not persist the
    failed submission.
```

This is the test that ES-03 from the requirement coverage map (Step 3, Day 1 assignment) was always pointing at. It was Not Covered before. This is what covering it looks like.

---

*Submitted by Deepti · Phase 3 Day 3 · Assignment 3 · Steps 1, 2 & 4 · June 2026*
*(Step 3 — PR #28 analysis to be added once files are shared)*

---

## Step 3 — PR #28: Refactor Member Task Detail UI

### What `MemberTaskDetail.tsx` Now Displays

Before PR #28, the admin member detail page showed raw task data. After the refactor, `MemberTaskDetail.tsx` is a fully interactive accordion-based view that shows an admin the following for each member:

**Per phase (PhaseAccordion):**
- Phase number badge (order), phase title, and completion percentage calculated live from task statuses
- A timeline status badge — one of: Completed, On track, Behind, Overdue, Not started — calculated by calling `getPhaseStatus()` with the phase's stored `startDate`, `dueDate`, and the current time. This badge only appears if the phase has a timeline set.
- The accordion defaults to open if the phase has any progress (`pct > 0`), closed if the member has not started it
- A chevron arrow that toggles the phase open/closed on click

**Per task (TaskCard inside each phase):**
- Task title with a green checkmark icon if done, grey circle if not done
- A badge: "Done" (emerald) if SUBMITTED or COMPLETED, "Not done" (grey) for CHECKBOX pending, "Not submitted" (grey) for TEXT/LINK pending
- Completion timestamp: if done and `completedAt` is set, shows "Completed Jun 20, 2026 at 10:34am" (for CHECKBOX) or "Submitted Jun 20, 2026 at 10:34am" (for TEXT/LINK). Note: only CHECKBOX tasks set `completedAt` — TEXT/LINK tasks will always show no timestamp even when submitted.
- For LINK tasks: the submitted URL rendered as a clickable link with the protocol stripped for display (e.g. `→ academy.hubspot.com/cert/abc`)
- For TEXT tasks: the full submitted response rendered in quotes below the task title, with `whitespace-pre-wrap` so line breaks are preserved

**What this means for an admin:** For the first time, an admin can see not just whether a task is done, but *what* a member submitted, *when* they submitted it, and whether they are on track or falling behind their timeline — all in one view without any additional navigation.

---

### What `SendReminderButton.tsx` Does

`SendReminderButton` is a new client-side component that renders a "Send reminder" button on the admin member detail page. When an admin clicks it:

1. The button immediately shows "Sending…" and disables itself (`sending = true`)
2. A `POST` request is sent to `/api/reminders` with the body `{ userId: "<member's id>" }`
3. The `try/finally` block runs — in the `finally`, `sending` is set back to `false`
4. If the fetch succeeds, `sent` is set to `true` and the button label changes to "Sent!" and stays permanently disabled

**What is not handled:**
- There is no catch block — same silent error pattern as `TextSubmission.tsx`. If `/api/reminders` fails or is unreachable, `sent` remains `false`, the button re-enables, and the admin sees no error message.
- There is no check on `res.ok` — the `fetch` call does not inspect the response status. Even if the server returns a 500, `sent` is set to `true` and the button shows "Sent!" as if the reminder was sent successfully. This is a more severe version of the silent error — it shows a false success state on a failed request.
- The button cannot be reset once `sent` is `true` — there is no way for the admin to send a second reminder in the same session without refreshing the page.

---

### 3 Targeted Test Cases for PR #28

These three tests exist only because of what this PR introduced. None of them would have been relevant before the refactor.

---

**TC-PR28-01: Phase accordion shows correct timeline status badge based on current date and progress**

| Field | Detail |
|---|---|
| Why it only exists because of this PR | `getPhaseStatus()` was not called in the admin view before. The timeline status badge is new. |
| Precondition | Admin is logged in. Member A has a phase with: startDate = 2 weeks ago, dueDate = 2 weeks from now, 20% of tasks done. A second member B has the same phase dates but 80% of tasks done. |

Steps:
1. Log in as admin. Navigate to `/admin/members/<member-A-id>`.
2. Locate the Phase 1 accordion header.
3. Observe the timeline status badge next to the phase title.
4. Navigate to `/admin/members/<member-B-id>`.
5. Locate the Phase 1 accordion header.
6. Observe the timeline status badge.

**Expected result:** Member A's phase badge shows "Behind" (amber) — they are at 20% but time is 50% elapsed. Member B's phase badge shows "On track" (emerald) — they are at 80% with only 50% of time gone. If no timeline is set for the phase, no badge appears at all. Badge colour and label must match the `timelineStatusConfig` mapping exactly.

---

**TC-PR28-02: TEXT task response is visible to admin in full, exactly as submitted**

| Field | Detail |
|---|---|
| Why it only exists because of this PR | Task response content was not displayed in the admin view before this PR. `MemberTaskDetail` now renders `prog.response` in quotes. |
| Precondition | Member has submitted a TEXT task with a multi-line response: `"Line one\nLine two"`. Admin is logged in. |

Steps:
1. Log in as admin. Navigate to `/admin/members/<member-id>`.
2. Expand the phase accordion containing the TEXT task.
3. Locate the task card for the submitted TEXT task.
4. Observe whether the response is displayed.
5. Observe whether the line break between "Line one" and "Line two" is preserved.
6. Confirm the badge shows "Done" in emerald.

**Expected result:** The full response text is visible inside quotes below the task title. Line breaks are preserved (`whitespace-pre-wrap` is applied). The task shows the "Done" badge. If the task is not submitted, no response section renders. No truncation — the full text is shown regardless of length.

---

**TC-PR28-03: Send reminder button shows "Sent!" after click and cannot be clicked again — and correctly handles API failure**

| Field | Detail |
|---|---|
| Why it only exists because of this PR | `SendReminderButton` is a new component. The reminder flow did not exist before. |
| Precondition | Admin is logged in. Member detail page is open. Run this as two sub-tests: once with `/api/reminders` returning 200, once intercepted to return 500. |

**Sub-test A — success path:**

Steps:
1. Navigate to `/admin/members/<member-id>`.
2. Locate the "Send reminder" button in the member summary card.
3. Click "Send reminder".
4. Observe the button label during the request.
5. Observe the button label after the request completes.
6. Attempt to click the button again.

Expected: During request → button shows "Sending…" and is disabled. After success → button shows "Sent!" and remains permanently disabled. Clicking again has no effect.

**Sub-test B — failure path (current known gap):**

Steps:
1. Intercept POST to `/api/reminders` and force a 500 response (via DevTools or a test mock).
2. Click "Send reminder".
3. Observe the button label after the request completes.

Expected (what *should* happen): Button shows an error state — e.g. "Failed to send" — and re-enables so the admin can try again. No false "Sent!" confirmation.

Actual (current behaviour): Button shows "Sent!" and stays disabled even though the reminder was never sent. **This is a bug introduced by PR #28** — `setSent(true)` fires regardless of `res.ok` because there is no response status check. This test currently fails Sub-test B.

---

*Submitted by Deepti · Phase 3 Day 3 · Assignment 3 · Complete · June 2026*
