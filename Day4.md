# Day 4 — Assignment 2: Design Tests Like a Pro
**BetaninjasSEOLMS · Task Submission Feature · Deepti · June 2026**

---

## Step 1 — Boundary Value Analysis

### 1A — Admin Timeline: Start Date & Due Date Fields
Source: `/admin/timeline` · `page.tsx` (PhaseTimeline)
Rule: `dueDate > startDate`

The boundary sits where the two dates are equal. One day on either side defines all meaningful boundary values.

#### Boundary Test Cases — Date Fields

| Boundary Point | Input Values | Expected Behaviour |
|---|---|---|
| Same date (ON boundary) | Start: 2026-06-20 / Due: 2026-06-20 | INVALID — Due Date is not after Start Date. Inline error shown. Save blocked. |
| One day before boundary (BELOW) | Start: 2026-06-20 / Due: 2026-06-19 | INVALID — Due is before Start. Error shown. Save blocked. |
| One day after boundary (ABOVE — min valid) | Start: 2026-06-20 / Due: 2026-06-21 | VALID — Closest valid gap. Saves successfully. |
| Large valid gap | Start: 2026-06-20 / Due: 2026-12-31 | VALID — Saves successfully. No upper limit imposed by spec. |
| Start Date empty, Due Date filled | Start: (empty) / Due: 2026-06-21 | INVALID — Start Date is required. Field error shown. |
| Due Date empty, Start Date filled | Start: 2026-06-20 / Due: (empty) | INVALID — Due Date is required. Field error shown. |
| Both fields empty | Start: (empty) / Due: (empty) | INVALID — Both required. Both field errors shown. |
| Due Date = Start Date + 1 year | Start: 2026-01-01 / Due: 2027-01-01 | VALID — No upper limit in spec. Saves successfully. |

---

### 1B — TEXT Task Submission: Character Limit Finding

> **Finding:** No character limit is defined in the feature spec for the TEXT task textarea. The spec says only "a member cannot submit an empty TEXT response" — it sets a lower boundary (> 0 chars) but specifies no upper boundary.

| Boundary | Observation / Finding |
|---|---|
| Lower bound (defined) | Empty field — Submit button disabled. 0 characters = blocked. This boundary IS enforced. |
| Lower bound + 1 (min valid) | 1 character typed — Submit should enable. Boundary just above the minimum. |
| Practical upper bound (undefined) | No `maxlength` attribute specified in spec. A member could theoretically paste 100,000 characters. |
| Browser default limit | Without a `maxlength` on the textarea, browsers do not impose a limit. The DB field type (TEXT vs VARCHAR) determines the real ceiling. |
| What happens beyond limit | Unknown. If the DB column is VARCHAR(255), a 256-char submission may silently truncate or throw a 500 error with no user feedback. |

> **Risk:** If the textarea has no `maxlength` and the DB column has a hidden limit, a member could type a long response, click Submit, and see a silent failure or an unhandled server error. This is a missing boundary definition — should be raised with Prem as a spec gap.

#### Recommended Test Cases — Character Limit

| TC | Input | Boundary Type | Expected |
|---|---|---|---|
| BVA-T1 | 0 characters (empty) | Lower bound — invalid | Submit disabled. Cannot submit. |
| BVA-T2 | 1 character | Lower bound +1 — min valid | Submit enabled. Submission accepted. |
| BVA-T3 | 255 characters | Common VARCHAR upper bound | Submission accepted. Full text saved and visible on return. |
| BVA-T4 | 256 characters | One above common upper bound | Submission accepted OR error shown — no silent truncation. |
| BVA-T5 | 5000 characters (large paste) | Stress boundary | Submission accepted OR clear error. Must not cause unhandled server error. |

---

## Step 2 — Equivalence Partitioning

Applied to the full task submission form across all three task types (CHECKBOX, TEXT, LINK).

Every input that behaves the same way can be represented by one test case. Testing one member of a class gives equal confidence to testing all members.

| Class | Description | Example Input | Representative Test Case |
|---|---|---|---|
| Valid — CHECKBOX | Member clicks checkbox on a CHECKBOX task | Checkbox toggle click | EP-01: Click checkbox — expect task moves to Completed, progress updates. |
| Valid — TEXT | Non-empty string in textarea | "Completed keyword research using Ahrefs" | EP-02: Submit 50-char text — expect task moves to Submitted, text saved. |
| Valid — LINK | Well-formed URL with protocol and domain | `https://academy.hubspot.com/cert/abc123` | EP-03: Submit valid HTTPS URL — expect Submitted, URL persists on refresh. |
| Invalid — Empty TEXT | Blank textarea on a TEXT task | `""` (empty string) | EP-04: Leave textarea empty, attempt submit — expect Submit button disabled or blocked. |
| Invalid — Empty LINK | Blank field on a LINK task | `""` (empty string) | EP-05: Leave URL field empty, attempt submit — expect Submit blocked with error. |
| Invalid — Malformed URL | String that starts with a protocol but is not a real URL | `https://,` or `https://??` | EP-06: Submit `https://,` — expect inline error, submission blocked. **(Known bug — currently FAILS.)** |
| Invalid — Plain text in URL field | Text with no protocol | `hubspot certificate` | EP-07: Submit plain text without `https://` — expect inline error, blocked. |
| Invalid — Wrong protocol | ftp or javascript protocol in URL field | `ftp://files.example.com` | EP-08: Submit `ftp://` URL — expect error (only http/https valid for a certificate link). |
| Unexpected type — CHECKBOX as TEXT | Attempt to type text into a CHECKBOX task | Typed response where no textarea exists | EP-09: Navigate to a CHECKBOX task — confirm no textarea is rendered. Checkbox is the only interaction. |
| Boundary edge — single char TEXT | Minimum non-empty input | `"a"` | EP-10: Submit single character — expect accepted, counts as Submitted. |

> Equivalence partitioning covers all meaningful input classes with 10 test cases. Running all possible inputs would give no additional confidence — the classes are the insight.

---

## Step 3 — Decision Table: Task Submission

**4 conditions:**
1. User is authenticated
2. Task ID is valid
3. Status is a valid enum value
4. Text/Link field is non-empty

**Key rule:** Authentication is checked first. An unauthenticated request receives 401 before any other condition is evaluated.

| Authenticated | Task ID Valid | Status Valid | Text Non-Empty | Row | Expected Outcome |
|---|---|---|---|---|---|
| NO | N/A | N/A | N/A | 1 | **401 Unauthorized** — redirect to login. No further checks run. |
| YES | NO | N/A | N/A | 2 | **404 Not Found** — task does not exist. Submission blocked. |
| YES | YES | NO | N/A | 3 | **400 Bad Request** — invalid status enum value. Submission rejected. |
| YES | YES | YES | NO | 4 | **400 Bad Request** — empty TEXT/LINK field. Submit button disabled on client; blocked on server. |
| YES | YES | YES | YES | 5 | **200 OK** — submission saved. Task state updated. Progress recalculated. |

#### One Test Case Per Row

| Row | Condition | How to Test | Expected Result |
|---|---|---|---|
| 1 | Not authenticated | Log out. POST `/api/submit` directly with a valid task ID and text body. | 401 — redirected to login. No submission saved. |
| 2 | Invalid Task ID | Log in. Submit to `/api/submit` with `taskId: 99999` (non-existent). | 404 — error response. No state change. |
| 3 | Invalid status enum | Log in. Submit with `status: "BANANA"` (not in enum) via API. | 400 — rejected. Valid enum values: NOT_STARTED, IN_PROGRESS, SUBMITTED, COMPLETED. |
| 4 | Empty text field | Log in. Open a TEXT task. Leave textarea blank. Click Submit. | Submit button is disabled. If API hit directly with empty string — 400 returned. |
| 5 | All conditions pass | Log in. Open a valid TEXT task. Type a response. Click Submit. | 200 — task moves to Submitted. Progress updates. Text persists on refresh. |

> **Note:** The text-field condition (Row 4) uses N/A for CHECKBOX tasks — CHECKBOX has no text field. The non-empty condition only applies to TEXT and LINK types.

---

## Step 4 — State Transition Testing

### States Defined in Spec

| State | Meaning |
|---|---|
| NOT_STARTED | Member has not opened the task yet. |
| IN_PROGRESS | Member has opened the task but not submitted. |
| SUBMITTED | Member has submitted a TEXT or LINK response. |
| COMPLETED | Member has checked off a CHECKBOX task. |

---

### State Diagram 1 — CHECKBOX Task

CHECKBOX tasks use only two states: NOT_STARTED and COMPLETED. They skip IN_PROGRESS and SUBMITTED entirely.

```
                         check (one click)
  ┌──────────────┐ ─────────────────────────> ┌─────────────┐
  │ NOT_STARTED  │                             │  COMPLETED  │
  └──────────────┘ <───────────────────────── └─────────────┘
                        uncheck (one click)
```

**Valid transitions**

| From | To | Trigger |
|---|---|---|
| NOT_STARTED | COMPLETED | Member clicks checkbox (check) |
| COMPLETED | NOT_STARTED | Member clicks checkbox again (uncheck) |

**Invalid transitions (should not be possible)**

| From | To (Invalid) | Why It Is Blocked |
|---|---|---|
| NOT_STARTED | IN_PROGRESS | CHECKBOX tasks skip this state — no textarea to open. |
| NOT_STARTED | SUBMITTED | SUBMITTED is for TEXT/LINK only. API should reject this for a CHECKBOX task. |
| COMPLETED | SUBMITTED | SUBMITTED is not a valid state for CHECKBOX type. |

---

### State Diagram 2 — TEXT and LINK Tasks

TEXT and LINK tasks use all four states. Opening a task automatically triggers the NOT_STARTED → IN_PROGRESS transition.

```
  ┌──────────────┐  open task   ┌─────────────┐  submit   ┌───────────┐
  │ NOT_STARTED  │ ──(auto)───> │ IN_PROGRESS │ ────────> │ SUBMITTED │
  └──────────────┘              └─────────────┘           └───────────┘
                                      ^                         |
                                      └─────── revise ──────────┘
                                               (resubmit)
```

**Valid transitions**

| From | To | Trigger |
|---|---|---|
| NOT_STARTED | IN_PROGRESS | Member opens the task for the first time (automatic). |
| IN_PROGRESS | SUBMITTED | Member submits a valid TEXT or LINK response. |
| SUBMITTED | IN_PROGRESS | Member edits their response before resubmitting. |
| IN_PROGRESS | IN_PROGRESS | Member returns later without submitting (state persists). |

**Invalid transitions (should not be possible)**

| From | To (Invalid) | Why It Is Blocked / Should Be Tested |
|---|---|---|
| NOT_STARTED | SUBMITTED | Member should not skip directly to Submitted without first opening the task. Key test below. |
| NOT_STARTED | COMPLETED | COMPLETED is for CHECKBOX only. API must reject this for TEXT/LINK types. |
| SUBMITTED | NOT_STARTED | Reverting to Not Started after submission is not a defined operation. Should be blocked. |
| IN_PROGRESS | COMPLETED | COMPLETED is CHECKBOX-only. Sending this status for a TEXT task via API should return 400. |

---

### Key Test: Can You Skip from NOT_STARTED to COMPLETED on a TEXT Task?

> **This is the most important invalid transition to test.** The spec defines COMPLETED as a CHECKBOX-only state. A direct API call attempting to set a TEXT task from NOT_STARTED to COMPLETED — without opening it (IN_PROGRESS) or submitting text (SUBMITTED) — should be rejected. If accepted, a member could mark a TEXT task done without ever writing anything.

#### State Transition Test Cases

| TC | Scenario | How to Test | Expected Result |
|---|---|---|---|
| ST-01 | CHECKBOX: NOT_STARTED → COMPLETED (valid) | Open a CHECKBOX task. Click the checkbox. | Task moves to Completed. Progress updates. One click. |
| ST-02 | CHECKBOX: COMPLETED → NOT_STARTED (valid) | On a Completed CHECKBOX task, click checkbox again. | Task returns to Not Started. Progress decreases. |
| ST-03 | TEXT: NOT_STARTED → IN_PROGRESS (auto) | Open a TEXT task for the first time. | Task state automatically moves to In Progress on open. No action needed. |
| ST-04 | TEXT: IN_PROGRESS → SUBMITTED (valid) | Type a response and click Submit. | Task moves to Submitted. Response saved. |
| ST-05 | TEXT: SUBMITTED → IN_PROGRESS (revise) | On a Submitted task, edit the text and click Submit again. | Returns to In Progress during edit, back to Submitted on resubmit. New text replaces old. |
| ST-06 | TEXT: NOT_STARTED → SUBMITTED (INVALID — skip) | Via API: POST `/api/submit` with a NOT_STARTED TEXT task and `status: SUBMITTED`. | Should be blocked. If the system has not first moved to IN_PROGRESS, this transition should be rejected. |
| ST-07 | TEXT: NOT_STARTED → COMPLETED (INVALID — wrong state for type) | Via API: POST `/api/submit` with `taskType: TEXT` and `status: COMPLETED`. | 400 Bad Request. COMPLETED is CHECKBOX-only. API must enforce type-state compatibility. |

---

## Step 5 — Testing Levels Mapped to Task Submission

| Level | What Is Tested Here | Example for Task Submission |
|---|---|---|
| **Unit Testing** | Individual functions and components in isolation, with mocked dependencies. | The URL validation function tested alone: pass in `https://,` and assert it returns false. Test the progress calculation formula (`tasks done ÷ total × 100`, rounded) with every boundary value, mocking no real DB. |
| **Integration Testing** | How multiple parts work together — the API route + the database + the auth middleware. | A real POST to `/api/submit` with a valid session token and a valid task ID: assert the DB row is written, the task status column updates, and the response returns 200 with the correct body. |
| **System Testing** | The full end-to-end flow in a production-like environment, as a real user would use it. | Log in as a member, navigate to Phase 1 Task 30, type a response, click Submit, return to the phase page, confirm the progress percentage has updated, refresh the page, confirm the response persists. |
| **Acceptance Testing** | Does the feature do what the business and users actually need — not just what the spec says. | A real team member (not QA) completes 3 tasks across two phases and confirms their progress bar matches what they expect, their submitted text is exactly what they typed, and nothing is lost on refresh or logout. |

> Each level catches different bugs. Unit catches logic errors early and cheaply. Integration catches contract mismatches between layers. System catches environment and flow issues. Acceptance catches assumption gaps — things the spec got wrong about what the user actually needs.

---

*Submitted by Deepti · Day 4 Assignment 2 · June 2026*
