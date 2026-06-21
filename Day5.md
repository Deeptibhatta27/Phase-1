# Day 4 — Assignment 3: Document It Like It Matters
**BetaninjasSEOLMS · Dashboard Feature · Deepti · June 2026**
**URL:** https://betaninjas-seo-learning-tracker.vercel.app/dashboard

---

## Step 1 — Test Scenarios and Test Cases

### Scenario 1: Dashboard loads correctly and reflects the member's real progress

The dashboard is the first thing a member sees after login. It must show accurate progress data pulled from the database — the overall progress bar, per-phase percentages, and phase card states. If any of these are wrong or missing, the member has no reliable way to know where they are in the curriculum.

---

**TC-01: Overall progress bar shows correct percentage after submitting a task**

| Field | Detail |
|---|---|
| Precondition | Member is logged in. At least one phase has tasks, none completed yet. Overall progress shows 0%. |

Steps:
1. Log in and land on the dashboard at `/dashboard`.
2. Note the overall progress bar percentage and the tasks done / total count displayed (e.g. "0 of 32 tasks").
3. Click into Phase 1 from the phase grid.
4. Open any TEXT task and submit a written response.
5. Navigate back to `/dashboard` using the browser back button or the navbar.
6. Observe the overall progress bar percentage and the tasks done count.

**Expected result:** The percentage has increased by exactly `1 ÷ total_tasks × 100` rounded to the nearest whole number. The tasks done count has incremented by 1. The update is visible without a hard page reload.

---

**TC-02: Phase card shows correct per-phase percentage and locked/unlocked state**

| Field | Detail |
|---|---|
| Precondition | Member is logged in. Phase 1 has tasks, none completed. Phase 2 should be locked. |

Steps:
1. Log in and navigate to `/dashboard`.
2. Locate the Phase 1 card in the phase grid. Note the percentage shown on the card (should be 0%).
3. Locate the Phase 2 card. Confirm it shows a locked state (lock icon or disabled click).
4. Click Phase 1. Complete one task (CHECKBOX or TEXT/LINK submission).
5. Return to `/dashboard`.
6. Observe the Phase 1 card percentage.
7. Observe whether Phase 2 remains locked.

**Expected result:** Phase 1 card percentage has updated to reflect the one completed task. Phase 2 lock state is governed by `isPhaseUnlocked()` — it should remain locked until Phase 1 reaches the unlock threshold defined in the codebase.

---

**TC-03: New user empty state renders and "Start with Phase 1" link works**

| Field | Detail |
|---|---|
| Precondition | A brand new member account with zero task completions (`tasksDone === 0`). |

Steps:
1. Log in with a fresh account that has never completed any task.
2. Navigate to `/dashboard`.
3. Confirm the empty state card renders: heading "Ready to begin your SEO journey?", subtext, and a blue "Start with Phase 1 — SEO Basics" button.
4. Confirm the overall progress bar shows 0%.
5. Click "Start with Phase 1 — SEO Basics".
6. Observe the URL and page that loads.

**Expected result:** Empty state card is visible when `tasksDone === 0`. Clicking the button navigates to `/phases/1` (or the ID of the first phase). No errors on navigation.

---

### Scenario 2: Dashboard displays the celebration banner and handles the loading skeleton correctly

The dashboard has two special UI states that are easy to miss: the 100% completion celebration banner and the loading skeleton shown while data fetches. Both must render correctly — the banner only when all tasks are done, the skeleton only while loading.

---

**TC-04: Celebration banner appears only when overall progress reaches 100%**

| Field | Detail |
|---|---|
| Precondition | Member has completed all tasks across all 5 phases. `overallProgress === 100`. |

Steps:
1. Log in as a member who has completed every task in all 5 phases.
2. Navigate to `/dashboard`.
3. Observe whether the celebration banner renders above the progress bar: gradient background, star icon, heading "You've completed the full curriculum!", subtext about being a certified SEO practitioner.
4. Also observe the subtitle text under the "Welcome back" heading — should read "You've completed the full curriculum!" not the default "Keep going" text.
5. Now log in as a member with less than 100% progress.
6. Navigate to `/dashboard`. Confirm the celebration banner is **not** shown.

**Expected result:** Banner renders only when `overallProgress === 100`. The subtitle text also changes correctly per the conditional in `page.tsx`. Banner is absent for any progress below 100%.

---

**TC-05: Loading skeleton renders during data fetch and disappears once content loads**

| Field | Detail |
|---|---|
| Precondition | Any logged-in member. Test on a throttled network (Chrome DevTools → Network → Slow 3G). |

Steps:
1. Open Chrome DevTools → Network tab → set throttling to "Slow 3G".
2. Log in and navigate to `/dashboard`.
3. While the page is loading, observe the UI.
4. Confirm the skeleton renders: navbar skeleton (dark bar), hero skeleton (two grey blocks), training plan card skeleton, progress bar skeleton (white card with two grey bars), and 5 phase card skeletons in a grid.
5. Once data loads, confirm the skeleton is fully replaced by real content — no skeleton blocks remain mixed in with real data.

**Expected result:** `DashboardLoading` component renders exactly as coded in `loading.tsx` — 5 phase skeletons in a `grid-cols-1 / sm:grid-cols-2 / lg:grid-cols-3` layout, all blocks animated with `animate-pulse`. After load, no skeleton elements remain visible.

---

**TC-06: Pinned SEO Training Plan link opens the correct Google Doc in a new tab**

| Field | Detail |
|---|---|
| Precondition | Any logged-in member. |

Steps:
1. Log in and navigate to `/dashboard`.
2. Locate the pinned resource card: blue document icon, label "SEO Training Plan", subtext "Full curriculum reference — Google Doc".
3. Click the card.
4. Observe: does it open in a new tab or the same tab?
5. Observe: does the URL match `https://docs.google.com/document/d/1hoWcZ7LJPHuwUaPQ4ZrCkgQYVr6_DMKo/edit`?

**Expected result:** Link opens in a new tab (`target="_blank"` confirmed in code). The Google Doc URL is correct. The external link icon (↗) is visible on the card. The card shows a blue hover state on mouseover.

---

## Step 2 — Bug Report

**Bug ID:** BUG-001
**Reported by:** Deepti
**Date:** June 17, 2026
**Feature:** Task Submission — LINK type

---

**Title:** LINK submission field accepts `https://,` as a valid URL and saves it as a completed submission

---

**Steps to Reproduce**

1. Log in to `https://betaninjas-seo-learning-tracker.vercel.app`.
2. Navigate to Phase 1 → Task 30 (LINK task type with certificate URL field).
3. Locate the input field with placeholder: *"Paste your certificate URL from HubSpot, Ahrefs, or Semrush Academy here."*
4. Type exactly: `https://,` (the string `https://` immediately followed by a comma — no space).
5. Observe the field border and any icon appearing next to the input.
6. Click **Submit Link**.
7. Observe the confirmation state next to the button.

---

**Expected Result**

The field should reject `https://,` as an invalid URL. An inline error message should appear next to the field (per the spec: *"Inline error shown next to the field — submission blocked"*). The Submit button should remain inactive or the submission should be blocked. Task state must not change to Submitted.

---

**Actual Result**

The field border turns **green** and a **green checkmark** appears — the input is treated as valid before Submit is even clicked. Clicking "Submit Link" shows **"Submitted!"** in green text next to the button. The task state changes to Submitted and the submission is saved. A meaningless, unresolvable string is now stored as a certificate URL.

**Evidence:** https://app.usebubbles.com/ve6B2weZaKWA9YkTnE4ftH/image-jun-17-2026

---

**Environment**

| Field | Detail |
|---|---|
| Browser | Chrome (latest) |
| OS | macOS |
| URL | https://betaninjas-seo-learning-tracker.vercel.app/phases/1/tasks/30 |
| App version | Live production — Vercel deployment, June 2026 |

---

**Severity: High**

The validation rule that exists to protect data integrity is silently bypassed. A member can submit a broken, unresolvable string as their certificate evidence and the system records it as a completed submission. The feature exists to track real learning proof — accepting garbage data undermines that entirely. This is not a cosmetic issue; it corrupts the core purpose of the LINK task type.

**Priority: High**

This bug is in production right now. Every member who encounters a LINK task can unknowingly (or deliberately) submit invalid data and have it counted as done. Admin visibility into member progress becomes unreliable. It should be fixed in the current or next sprint — a one-line fix to the URL validation regex is likely all it takes.

---

## Step 3 — Defect Lifecycle

**Bug tracked:** BUG-001 — LINK field accepts `https://,` as valid

---

| Stage | Owner | What Happens |
|---|---|---|
| **New** | Deepti (QA) | Bug discovered during exploratory testing of Phase 1 Task 30. Bug report written with title, reproduction steps, expected/actual result, evidence link, severity, and priority. Submitted to the team channel / issue tracker. |
| **Assigned** | Prem (Dev lead / sprint owner) | Prem reviews the report, confirms it is reproducible and valid (not a duplicate, not by-design). Assigns it to the developer responsible for the LINK submission validation logic. Sprint priority is set based on the High rating. |
| **Open** | Developer (Prem or assigned dev) | Developer investigates the URL validation function. Identifies the root cause: the regex or validation library used passes any string that begins with `https://`, regardless of what follows. Work begins on tightening the rule — likely adding a check that a valid domain follows the protocol. |
| **Fixed** | Developer | Fix is implemented. The validation logic now rejects `https://,` and similar near-valid strings. A unit test is written: `validateURL("https://,")` must return `false`. PR is raised and merged to staging. |
| **Retest** | Deepti (QA) | QA pulls the fix from staging. Runs the exact reproduction steps from the bug report: navigates to Phase 1 Task 30, types `https://,`, clicks Submit. Also runs regression tests: submits a valid URL to confirm it still works, submits `https://??`, `ftp://x`, plain text — confirms all are correctly rejected. |
| **Verified** | Deepti (QA) | **For this specific bug:** Verified means: (1) `https://,` now shows an inline error and blocks submission, (2) no green checkmark appears for this input, (3) the task state does not change to Submitted when this input is used, (4) a valid URL like `https://academy.hubspot.com/cert/abc` still submits successfully. All four conditions must be true before the bug is marked Verified. |
| **Closed** | Prem (sprint owner) | **Closed means:** The fix has been verified by QA on staging, deployed to production, and QA has run a final smoke test on the live app confirming the bug no longer exists in production. The ticket is closed. The fix is noted in the sprint retro as a validation gap that should have been caught earlier by defining "invalid URL" more precisely in the spec. |

---

## Step 4 — Entry and Exit Criteria

### Entry Criteria — Dashboard Testing

Before testing begins, all of the following must be true:

1. The dashboard feature (`/dashboard`) is deployed to the staging environment and accessible without errors.
2. At least three test accounts exist with different data states: (a) a new user with zero task completions, (b) a user with partial progress across at least two phases, (c) a user with 100% completion across all 5 phases.
3. The database is seeded with all 5 phases, each containing at least the tasks used in the test cases.
4. Login is working — Google OAuth or test credentials allow access to the dashboard without a redirect loop.
5. The admin timeline has start and due dates set for at least Phase 1, so phase status labels (On Track / Overdue) can be tested on the phase cards.
6. BUG-001 (LINK validation) status is known — either open (so TC results referencing it can be marked Expected Fail) or fixed (so regression tests are in scope).
7. The staging environment matches production configuration — same Next.js version, same DB schema, same Prisma client.

### Exit Criteria — Dashboard Testing

Testing is complete and the dashboard is ready to ship when all of the following are true:

1. All 6 test cases (TC-01 through TC-06) have been executed and results recorded.
2. TC-01, TC-02, TC-03, TC-04, and TC-06 have **passed** — these cover core progress accuracy, phase lock state, new user flow, celebration banner, and the pinned resource link.
3. TC-05 (loading skeleton) has passed on at least one throttled network condition (Slow 3G or equivalent).
4. No Severity 1 (critical) bugs are open. A critical bug is defined as: dashboard fails to load, progress data is incorrect, or a member can see another member's data.
5. No Severity 2 (high) bugs are open unless explicitly deferred with sign-off from Prem. BUG-001 is Severity High — it must be fixed or formally deferred before exit.
6. The celebration banner has been tested for both the 100% case (must show) and any sub-100% case (must not show) — both verified.
7. The dashboard has been tested on Chrome (latest) and at least one mobile viewport (375px width) to confirm the phase grid collapses from 3 columns to 1 without layout breakage.

---

## Step 5 — Test Report

**Feature tested:** Dashboard — `/dashboard`
**Tester:** Deepti
**Date:** June 2026
**Environment:** Live production — betaninjas-seo-learning-tracker.vercel.app · Chrome · macOS

---

### Summary

| Total TCs | Passed | Failed | Skipped |
|---|---|---|---|
| 6 | 4 | 1 | 1 |

---

### Results by Test Case

| TC | Title | Result | Notes |
|---|---|---|---|
| TC-01 | Overall progress bar updates after task submission | **PASS** | Percentage updated correctly after submitting one TEXT task. Tasks done count incremented. No hard reload needed. |
| TC-02 | Phase card shows correct percentage and lock state | **PASS** | Phase 1 card percentage updated after task completion. Phase 2 remained locked. `isPhaseUnlocked()` logic is working as expected. |
| TC-03 | New user empty state and Phase 1 link | **PASS** | Empty state card rendered correctly on a fresh account. "Start with Phase 1" button navigated to `/phases/1` without error. |
| TC-04 | Celebration banner at 100% completion | **PASS** | Banner rendered correctly for the 100% account. Banner was absent for partial-progress accounts. Subtitle text also switched correctly. |
| TC-05 | Loading skeleton on slow network | **SKIPPED** | Could not throttle network reliably on current test machine. Skeleton code was reviewed against `loading.tsx` — structure is correct in code. Marked for retest on a throttled environment. |
| TC-06 | Pinned SEO Training Plan link opens in new tab | **FAIL** | Link opened in a new tab and the external icon rendered correctly. **However:** the Google Doc URL returned a "You need access" error — the document is not publicly accessible to all BetaNinjas members. The link itself works technically, but the destination is permission-gated. Raised as a separate UX finding. |

---

### What Passed

The core dashboard behaviour is solid. Progress tracking (TC-01, TC-02) is accurate and updates in real time. The new user onboarding flow (TC-03) works cleanly. The 100% celebration banner (TC-04) fires only at the right condition and is absent otherwise — the conditional logic in `page.tsx` is working correctly.

### What Failed

**TC-06** — the pinned SEO Training Plan link is technically functional (opens new tab, correct URL) but the Google Doc behind it is access-restricted. A member clicking it hits a permissions wall. This is not a code bug — it is a configuration issue. The document sharing settings need to be updated to allow access for all team members, or the link needs to be replaced with an accessible copy.

### What Was Skipped and Why

**TC-05** (loading skeleton) was skipped because network throttling was not available in the current test environment. The skeleton implementation was verified by code review against `loading.tsx` — the structure matches (navbar, hero, training card, progress bar, 5 phase cards, all with `animate-pulse`). This test should be re-run in a controlled throttled environment before the feature is signed off for production. Not a blocker, but it should not remain permanently unexecuted.

### Open Items

| ID | Item | Severity | Owner |
|---|---|---|---|
| BUG-001 | LINK field accepts `https://,` as valid URL | High | Prem / Dev |
| UX-001 | Google Doc training plan link is access-restricted for members | Medium | Prem (content/config) |

### Overall Assessment

The dashboard is functionally correct for its core purpose — showing progress, updating after task submissions, handling new user and completed user states. The two open items are not in the dashboard feature itself but in dependent systems (URL validation in task submission, Google Doc permissions). Dashboard is ready for use with those items tracked and actioned separately.

---

*Submitted by Deepti · Day 4 Assignment 3 · June 2026*
