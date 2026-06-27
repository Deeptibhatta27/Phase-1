# Day 5 — Assignment 2: Coverage Strategy & Confidence
**BetaninjasSEOLMS · Phase 3 Day 2 · Deepti · June 2026**

---

## Step 1 — Coverage Strategy

Based on the risk matrix from Day 1. The goal is not a percentage — it is a deliberate decision about where to spend testing effort and what can never be skipped.

---

### 🔴 High Risk Features — Deep Coverage

**Rule: these features are tested at every level. Nothing that touches data integrity or access control is skipped.**

---

#### Task Submission (TEXT, LINK, CHECKBOX)

This is the core action the app exists to support. If submission is broken, the entire product fails its purpose.

**What deep coverage looks like:**

- **Unit:** URL validation function tested with every equivalence class — valid HTTPS, malformed (`https://,`, `https://??`, `://nothing`), missing protocol, wrong protocol (`ftp://`), empty string. The `isDone` helper tested with every possible status string including unexpected values.
- **Integration:** POST to `/api/submit` with a real DB — assert the exact row written, the exact status stored, the exact response returned. Test with valid session, expired session, missing session. Test each task type (CHECKBOX, TEXT, LINK) separately at the API layer.
- **System:** Full login → navigate → submit → return to phase page → confirm progress updated → refresh → confirm data persists. Run for all three task types.
- **Regression:** Every time a change is made to the submission API or validation logic, the URL validation unit tests and the integration test for the happy path must pass before merge.

**Scenarios that can never be skipped:**
- `https://,` and similar near-valid strings must be rejected (BUG-001 — currently failing)
- Empty TEXT submit must be blocked at both client and server level
- Submission data must persist after a full page refresh
- A failed submission must never show a success state
- Revision must replace, not append, the previous response

---

#### Progress Tracking (`calcPhaseProgress`, `calcOverallProgress`, `isDone`)

Every number a member and admin sees comes from these functions. A silent calculation error gives every user wrong data with no visible signal.

**What deep coverage looks like:**

- **Unit:** `calcPhaseProgress` tested with: zero tasks (expect 0), all tasks done (expect 100), one of three done (expect 33), two of three done (expect 67). `calcOverallProgress` tested with multiple phases, mixed completion states. `isDone` tested with every status string: SUBMITTED → true, COMPLETED → true, IN_PROGRESS → false, NOT_STARTED → false, undefined → false, garbage string → false.
- **Boundary:** Rounding tested explicitly — 1 of 3 tasks done must return 33 not 34, 2 of 3 must return 67 not 66. `Math.round` at the 0.5 boundary.
- **Integration:** After submitting a task via the API, query the DB and assert the stored status. Then call `calcPhaseProgress` with the updated map and assert the exact expected integer.
- **System:** Submit tasks and confirm the displayed % on the phase page matches the formula output exactly.

**Scenarios that can never be skipped:**
- Zero tasks in a phase — must return 0, not divide-by-zero
- All tasks done — must return exactly 100, not 99 or 100.0
- Mixed SUBMITTED and COMPLETED statuses — both must count as done
- Progress update must be visible without a page reload

---

#### Login / Authentication

If login breaks entirely, no one can use the product. If auth is bypassed, data exposure is the result.

**What deep coverage looks like:**

- **Unit:** Auth helper functions tested in isolation — session validation, role extraction, redirect logic.
- **Integration:** Middleware tested with: valid session → allow through, missing session → redirect to login, member session on admin route → redirect to dashboard, admin session on admin route → allow through.
- **System:** Full browser login flow with valid credentials, invalid password, invalid email format, empty fields. Also: logged-in user trying to access `/login` again must be redirected.
- **Security:** Attempt to access `/dashboard` and `/admin` routes without a session cookie — must redirect, not serve content.

**Scenarios that can never be skipped:**
- Non-admin member must not reach any `/admin` route
- Logged-in user revisiting `/login` must be redirected to `/`
- Session expiry must redirect to login, not serve stale content

---

#### Phase Lock State (`isPhaseUnlocked`)

Controls curriculum progression. Currently hardcoded `return true` — all phases open to all members. If this ships without the real logic, the structured learning path is meaningless.

**What deep coverage looks like:**

- **Unit:** `isPhaseUnlocked` tested with: Phase 1 (always unlocked), Phase 2 with Phase 1 incomplete (must return false), Phase 2 with Phase 1 fully complete (must return true), Phase 2 with Phase 1 partially complete (must return false), missing previous phase (must return true as fallback).
- **Integration:** Navigate to Phase 2 as a member with zero Phase 1 completions — phase card must show locked state.
- **Regression:** This function must have a unit test that asserts `false` for a locked phase *before* the hardcoded `return true` is ever removed. The test must fail with the current code — that failure is proof the fix is needed.

**Scenarios that can never be skipped:**
- Phase 2 locked when Phase 1 is 0% complete
- Phase 2 locked when Phase 1 is 99% complete (not fully done)
- Phase 2 unlocked only when Phase 1 is exactly 100%
- Phase 1 always unlocked regardless of any other state

---

### 🟡 Medium Risk Features — Reasonable Coverage

**Rule: happy path + one meaningful error case + one edge case. Not every combination, but enough to confirm the feature works and fails gracefully.**

---

#### Dashboard

Display layer only — does not write data. Bugs are visible immediately.

**Minimum that gives confidence:**
- Overall progress bar shows correct % for a member with known progress state (1 system test)
- Phase cards render with correct per-phase % and correct lock/unlock visual state (1 system test)
- New user empty state renders when tasksDone = 0 (1 system test)
- Celebration banner appears only at 100% and is absent below 100% (1 system test)
- Skeleton loading state renders during data fetch — verified once on throttled network

What can be skipped: testing every possible progress % value, testing the dashboard for every edge case in progress calculation (those belong at the unit level in `calcPhaseProgress`).

---

#### Admin View

Affects one or two admin users only. No member data is at risk.

**Minimum that gives confidence:**
- Admin can see the task-level status for a specific member (1 system test — currently Not Covered)
- Admin view shows accurate data after a member submits a task (1 integration test asserting the query returns the updated status)
- Non-admin member cannot reach the admin view (already covered by auth integration tests)

What can be skipped: testing every combination of member × task × status. One known data state with a verified display is sufficient.

---

#### Phase Page (task list view)

Renders the task list. A bug here is visible and does not corrupt data.

**Minimum that gives confidence:**
- Tasks render in correct order for a phase (1 system test)
- Clicking a task navigates to the correct task page (1 system test)
- Phase progress % on the phase page updates after a submission without reload (already covered in submission tests)

What can be skipped: testing every phase separately. One phase with known task data is representative.

---

#### Timeline / Status Labels (`getPhaseStatus`)

Wrong labels are confusing but do not corrupt data. Medium risk but the function has zero tests — that makes it feel more urgent than its risk rating.

**Minimum that gives confidence:**
- One unit test per path: COMPLETED, NOT_STARTED, OVERDUE, ON_TRACK, BEHIND (5 unit tests — see Step 2 coverage analysis from Day 1)
- One system test confirming the correct label appears on a phase card for a known timeline state

What can be skipped: testing every combination of date arithmetic. The unit tests on `getPhaseStatus` cover all paths; system tests do not need to replicate them.

---

#### Zod Validation Schemas

Simple but their failure has downstream consequences.

**Minimum that gives confidence:**
- `progressSchema`: test with valid input (passes), invalid taskId (negative int — fails), invalid status enum (fails), invalid URL in link field (fails)
- `TimelineUpdateSchema`: test with dueDate > startDate (passes), dueDate = startDate (fails), dueDate < startDate (fails), non-datetime string (fails)
- `planPhaseUpdateSchema` and `planTaskUpdateSchema`: test with all fields filled (passes), one field empty (fails)

These are pure unit tests — no UI, no DB, just schema.parse(input) and assert.

---

### 🟢 Low Risk Features — Light Coverage

**Rule: one smoke test that the feature exists and does not throw an error. Nothing more.**

---

#### Pinned Resource Link / Static UI

**Absolute minimum:**
- The SEO Training Plan link renders on the dashboard and opens a new tab. One manual check. No automation needed unless the URL changes frequently.
- If the Google Doc permission issue (UX-001) is fixed, one check that the document is accessible to a logged-in member.

What can be skipped: everything else. The link is a static `href`. There is nothing to break at the code level.

---

## Step 2 — Mobile Coverage Thinking

BetaninjasSEOLMS is a web app, but the coverage thinking below applies directly if it were a mobile app — and partially applies now for responsive web testing on phones.

---

### Device Coverage Matrix

**Principle:** Test the devices the team actually uses, plus the minimum to cover OS/browser diversity. Not every device — the one that represents a class.

| Device | OS | Browser | Priority | Reason |
|---|---|---|---|---|
| iPhone 15 Pro | iOS 17 | Safari | 🔴 High | Most likely device in use on the BetaNinjas team. Safari on iOS has distinct rendering and input behaviour — especially for form fields and date inputs. |
| iPhone 13 / 14 | iOS 16 | Safari | 🟡 Medium | Previous generation iOS still widely used. iOS 16 has known differences in `input[type=date]` and focus/blur behaviour that affect form validation. |
| Samsung Galaxy S23 | Android 14 | Chrome | 🔴 High | Leading Android device. Chrome on Android is the dominant non-iOS browser. Must cover the submission forms and progress display. |
| Samsung Galaxy A54 | Android 13 | Chrome | 🟡 Medium | Mid-range Android — representative of lower-end hardware on the team or for future members. Slower JS execution affects timing-sensitive UI like progress bar updates. |
| iPad (10th gen) | iPadOS 17 | Safari | 🟡 Medium | Tablet viewport — the phase grid switches between 1, 2, and 3 columns. iPad sits in the `sm` breakpoint zone where layout bugs often hide. |
| Desktop Chrome (macOS) | macOS 14 | Chrome 125+ | 🔴 High | Primary development and testing environment. Already covered in all existing test cases. |
| Desktop Safari (macOS) | macOS 14 | Safari 17 | 🟡 Medium | Safari on desktop has different form autofill, date picker, and focus behaviour compared to Chrome. Admin timeline date fields are a specific risk. |
| Windows 11 | Windows 11 | Chrome / Edge | 🟡 Medium | At least one Windows test covers the team's non-Mac users. Edge is Chromium-based so one test covers both. |

**What can be skipped:** Older Android (< 12), non-mainstream browsers (Firefox mobile, Opera), tablets below iPad size. These represent less than 5% of likely usage and the risk does not justify the effort.

---

### Network Coverage Matrix

| Network Condition | Speed Profile | Priority | Features Most at Risk |
|---|---|---|---|
| Strong WiFi / Broadband | > 20 Mbps, < 50ms latency | 🟢 Baseline | All features. This is the happy path environment — all current tests run here. |
| 4G LTE (good signal) | ~10 Mbps, ~80ms latency | 🟡 Medium | Task submission, progress update. Slight delay but all requests should complete. Loading skeleton should appear briefly. |
| 3G (slow mobile) | ~1.5 Mbps, ~200ms latency | 🔴 High | Task submission, progress update, dashboard load. Submission requests may take 2–4 seconds. The UI must not show success before the server confirms. Loading skeleton must render cleanly. Phase card progress must not flash incorrect values. |
| Intermittent / Flaky connection | Packet loss 20–40% | 🔴 High | Task submission specifically. A partially sent submission that times out must not be shown as Submitted. Error state ES-03 (network error during submit) — currently Not Covered — is the critical test here. |
| Offline (no connection) | 0 connectivity | 🔴 High | Task submission, login. Member must see a clear error or the app must not allow submission. No ghost Submitted states. No silent data loss. |
| WiFi → 3G mid-session transition | Connection drops and recovers | 🟡 Medium | Submission in progress when connection drops. Does the form hold its content? Does the member see their input still in the field as the spec requires? |

---

### What Happens Feature by Feature Under Each Network Condition

**Task Submission under Slow 3G:**
The submission request takes longer to complete. The critical risk: if the UI shows "Submitted!" before the server responds, and the request then times out, a member believes their work was saved when it was not. The spec says explicitly: "UI does not show success — member sees their input still in the field" on network error. This must be tested under simulated 3G using Chrome DevTools throttling.

**Task Submission offline:**
The POST to `/api/submit` fails immediately. The UI must not transition the task state to Submitted. The member must see their typed response still in the field. This is currently Not Covered — there is no test for this scenario at any network condition.

**Dashboard load on slow 3G:**
The loading skeleton (`loading.tsx`) must render before data arrives — this is the exact scenario TC-05 was skipped for in Assignment 3. On slow 3G, the skeleton is visible for 2–4 seconds. If it does not render, the member sees a blank white screen. Tested by throttling network in DevTools before navigating to `/dashboard`.

**Progress update on flaky connection:**
After submission, the progress percentage updates via a client-side recalculation. If the submission POST fails silently, the progress bar might update in the UI while the DB was never written — creating a phantom progress state that disappears on refresh. This is the most dangerous network failure mode and must be tested explicitly.

**Login on slow 3G:**
Login depends on Google OAuth redirect timing. On slow connections, the redirect chain (app → Google → callback → dashboard) may time out. The member must see a clear error, not a blank page or infinite loader.

**Admin timeline save on slow 3G:**
The admin saves date changes for a phase. On slow connection, the save request may not complete before the admin navigates away. Data loss risk — the dates revert to their previous values with no warning.

---

## Step 3 — The Gap Between Coverage and Confidence

### Not Covered Requirements — What Could Go Wrong

From the Day 1 requirement coverage map:

| Not Covered Requirement | What Could Go Wrong If Never Tested |
|---|---|
| **US-04 / PD-03 — Admin view of member task submissions** | An admin makes sprint decisions based on progress data that is silently wrong — a DB query bug or missing join means some members' submissions never appear, and no one knows until a member manually reports it. |
| **ES-03 — Network error during submission, UI must not show success** | A member submits a TEXT task on a slow connection, sees "Submitted!", closes their laptop, and their response was never saved — they discover the loss only when they return and find the task still In Progress with their text gone. |
| **ES-04 — Session expiry mid-submission** | A member spends 20 minutes writing a detailed TEXT response, their session expires silently, they click Submit, are redirected to login, and the entire response is lost — the spec says submission must not be saved, but nothing guarantees the response text is recoverable either. |
| **PD-04 (partial) — Progress updates without a page reload** | The progress bar on the phase page shows the old percentage after a submission — a member thinks their task did not save and submits again, creating duplicate state, or navigates away thinking the feature is broken when it is not. |

---

### Honest Evaluation of a Covered Test Case

**The test case: TC-02 — TEXT task, submit and revise a response**

The test steps are:
1. Open a TEXT task
2. Type a response
3. Click Submit
4. Confirm task shows Submitted
5. Refresh — confirm text is visible
6. Edit and resubmit
7. Confirm new text replaced old

**Is this proving the requirement, or just going through the steps?**

Steps 1–4 go through the steps. Opening the task, typing, clicking Submit, and checking the state label are actions — they confirm something happened, but "task shows Submitted" is a state label, not a data assertion. The label could show Submitted while the wrong text (or empty text) was stored.

Step 5 is where this test gets real confidence. Refreshing the page and confirming the submitted text is still visible proves the data actually reached the DB and was retrieved correctly. This step turns a walkthrough into a proof.

Step 7 is the most specific assertion: confirming the *new* text replaced the *old* one. This requires knowing what the old text was and asserting it is gone — not just that *some* text is present. If this step only checks that a response exists, it is coverage. If it checks that the response matches exactly what was typed in step 6 and that the text from step 2 is absent, it is confidence.

**The difference in one sentence:**

Coverage is confirming the task reached Submitted state. Confidence is confirming the exact text submitted is the exact text stored and the old text is gone.

---

### The One Test Case That Gives the Most Confidence

**TC-03 from Phase 2 Decision Table — Row 1: Unauthenticated POST to `/api/submit` returns 401**

Why this gives the most confidence:

It is the only test case that targets the server layer directly, bypasses the UI entirely, and makes a specific assertion about a security boundary — not just a functional outcome. The assertion is binary and unambiguous: either the server returns 401 and rejects the request, or it does not. There is no room for interpretation.

It also proves something that the UI can never prove: that the protection exists at the API level, not just because a button is not visible. A UI test that confirms a logged-out user cannot see the Submit button does not prove the API rejects a POST from an unauthenticated client. This test does. If someone builds a client that bypasses the UI and hits the endpoint directly, this test is the only one that catches it.

A test that asserts `response.status === 401` for an unauthenticated request is not going through steps — it is proving a security property.

---

## Step 4 — What Good Coverage Looks Like for BetaninjasSEOLMS

### What Is Currently Covered

From Phase 1 and Phase 2 combined (43 test cases):

- **Login flow** is reasonably covered — 5 system tests cover the UI paths, 2 integration tests cover the middleware redirects. It is the best-covered High-risk feature.
- **Task submission — happy paths** are covered at the system level for all three task types (CHECKBOX, TEXT, LINK). Submit, revise, and persist are all tested.
- **Error states for empty submission** are covered — empty TEXT is blocked at both client and server level in the test cases.
- **Progress display** is covered at the system level — percentage updates and dashboard rendering are tested through the UI.
- **State transitions** for all four task states are covered including the key invalid transitions (ST-06, ST-07) that test the API boundary.
- **BVA for date fields** gives reasonable boundary coverage for the admin timeline.

### The Most Important Gap

**`isPhaseUnlocked` has no test that asserts it should return `false` — and the function currently returns `true` for everything.**

This is the single highest-value missing test because:

1. The bug is already in production (hardcoded `return true`, real logic commented out)
2. It affects every member's experience of the curriculum — all phases are open right now
3. A single unit test with `expect(result).toBe(false)` for a member with zero Phase 1 completions would immediately fail and surface the bug
4. No system test currently catches it because the UI shows phases as unlocked — which matches the broken function's output

Fixing this gap costs one unit test. The confidence gain is disproportionately high.

---

### 5 Test Cases to Write Next — and Why

**TC-NEXT-1: `isPhaseUnlocked` — Phase 2 locked when Phase 1 is incomplete (Unit)**

```
Input:  phaseOrder = 2, Phase 1 tasks = [{id:1},{id:2}], progressMap = {}
Assert: result === false
```

Why: This is the Kristi bug. It exists right now in the codebase. This test fails immediately with the current code, proving the bug exists, and passes only when the real logic is restored. Highest return on one test.

---

**TC-NEXT-2: `calcPhaseProgress` — rounding at the exact 0.5 boundary (Unit)**

```
Input:  3 tasks, 1 done  → assert 33 (not 34)
Input:  2 tasks, 1 done  → assert 50
Input:  0 tasks          → assert 0 (not divide-by-zero)
```

Why: Progress numbers are the core output of the app. If rounding is wrong, every member sees incorrect percentages. This tests the formula the spec defines explicitly.

---

**TC-NEXT-3: Task submission on simulated slow 3G — no ghost success state (System)**

```
Steps:  Enable Chrome DevTools Slow 3G throttle.
        Open a TEXT task. Type a response. Click Submit.
        Immediately throttle to Offline before response returns.
Assert: "Submitted!" does NOT appear. Input text remains visible in the field.
        Task state remains IN_PROGRESS after reload.
```

Why: ES-03 is Not Covered and is the most dangerous real-world failure mode. A ghost success state means a member believes their work was saved when it was not. This directly violates a spec success metric.

---

**TC-NEXT-4: `progressSchema` — invalid URL in the link field rejected by Zod (Unit)**

```
Input:  { taskId: 1, status: "SUBMITTED", link: "https://," }
Assert: schema.parse() throws ZodError on the link field
```

Why: This is BUG-001 at the schema layer. The URL validation that fails in the UI may also fail in the Zod schema — or it may not, depending on how `z.string().url()` is implemented. This test determines whether the bug exists at validation time or only in the client-side UI. The fix may be one character in the schema.

---

**TC-NEXT-5: Admin member view shows accurate task-level status (System)**

```
Precondition: Member A has submitted Phase 1 Task 1 (TEXT) and completed Phase 1 Task 2 (CHECKBOX).
Steps:        Log in as admin. Navigate to the member view for Member A.
Assert:       Task 1 shows status SUBMITTED with the correct response text visible.
              Task 2 shows status COMPLETED.
              Task 3 (not started) shows NOT_STARTED.
```

Why: US-04 and PD-03 are both Not Covered. The admin view is the only place where data integrity is visible to a non-member. If the admin sees wrong data, sprint decisions are made on false information. This test would also have caught any DB query bug in the admin data pipeline.

---

### The Summary in One Paragraph

BetaninjasSEOLMS has reasonable system-level coverage of its visible flows — login, task submission happy paths, progress display, and state transitions are all tested at the UI level. The gaps are almost entirely at the unit and integration layers, in exactly the functions that carry the most risk: `isPhaseUnlocked` is hardcoded broken with no test to catch it, `calcPhaseProgress` and `calcOverallProgress` have no unit tests despite driving every number the product shows, `getPhaseStatus` has five untested paths, and the network failure error state (ES-03) has never been exercised. The test suite looks like a wide, shallow net — it catches things that are visibly broken, but it misses logic errors, data integrity failures, and security boundary violations that only surface when you test below the UI. The five test cases above target exactly those gaps, in order of how much confidence each one adds per minute of effort.

---

*Submitted by Deepti · Day 5 Assignment 2 · Phase 3 Day 2 · June 2026*
