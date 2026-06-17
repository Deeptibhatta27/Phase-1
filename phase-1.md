Day 2 Assignments
QA Theory & Practice  ·  June 16, 2026
Deepti  Bhatta

Assignment 1
1. The Bug — What It Is and How to Reproduce It
App: BetaninjasSEOLMS  ·  https://betaninjas-seo-learning-tracker.vercel.app
Page: Phase 1, Task 30 
Bug: The LINK submission field accepts 'https://,' as a valid URL and shows a green checkmark + 'Submitted!' confirmation — even though 'https://,' is not a real URL.

Steps to Reproduce
-Log in to betaninjas-seo-learning-tracker.vercel.app and navigate to Phase 1, Task 30.
-Locate the LINK submission field (placeholder: 'Paste your certificate URL from HubSpot, Ahrefs, or Semrush Academy here.').
-Type or paste: https://,  (the string 'https://' followed immediately by a comma)
-Observe: the field border turns green and a green checkmark appears — the input is treated as valid.
-Click 'Submit Link'.
Observe: 'Submitted!' appears in green next to the button. The submission is accepted and saved.

The spec says: A member cannot submit an invalid URL for a LINK task. The system should block this and show an inline error. Instead it shows a green checkmark and saves the submission.

Evidence: https://app.usebubbles.com/ve6B2weZaKWA9YkTnE4ftH/image-jun-17-2026


2. Which of the 7 Testing Principles It Relates To
Principle: Testing Shows Presence of Defects (not Absence)
This bug directly shows why testing shows presence of defects, not absence. The happy-path test submitting a real, well-formed URL like https://academy.hubspot.com/certificate/abc passes cleanly. If testing had stopped there, the feature would appear correct. It was only by testing a near-miss invalid input (a string that starts with a valid protocol but is not a real URL) that the defect surfaced.
It also connects to the principle of Exhaustive Testing is Impossible. No team will test every string a user might type into a URL field. That is precisely why boundary and equivalence partitioning techniques exist,      to deliberately target the edges (valid-looking but malformed inputs) without testing everything. 'https://,' sits exactly on that boundary: it begins like a URL, so a naive regex or starts-with check passes it through, but it resolves to nothing and is meaningless as a certificate link.
The practical lesson: passing the happy path gives false confidence. The defect is not in the obvious failure case (typing 'hello'); it is in the near-valid case that slips past weak validation.
3. Which SDLC Stage Could Have Caught It
Stage: Development : Unit / Integration Testing of URL Validation Logic
This bug most clearly should have been caught during development, when the URL validation function was written. The spec is explicit: a member cannot submit an invalid URL. That requirement implies a validation rule exists but the rule was implemented too loosely. A developer writing a unit test for the validation function would have included a test case like: does 'https://,' return invalid? It does not, and the test would have failed, catching the bug before it ever reached staging.
It could also have been caught in the QA / Test stage if the test plan for the LINK task type included an equivalence partition for 'structurally plausible but meaningless URLs' alongside the obvious bad cases (empty field, plain text, missing protocol). The spec calls out 'invalid URL' without defining what invalid means  which is itself a gap that a QA engineer reviewing the spec at the design stage could have flagged: what exactly counts as invalid? A regex? A resolvable domain? A specific format for certificate URLs?
4. Static or Dynamic — Which Would Have Found It First
Answer: Dynamic Testing Found It — Static Review Could Have Tightened the Spec
Dynamic testing found this bug. I had to open the live app, type an actual input, click the button, and observe the result. No amount of reading the spec would have reproduced the behaviour  the spec just says 'invalid URL shows inline error' without specifying what the validation logic should cover.

That said, static testing had a role to play upstream. If a QA engineer had reviewed the spec at the design stage and flagged: 'the spec says invalid URL but does not define the validation rules — does https://OK? Does https://,? Does a URL with no TLD?'  the developer would have been given a precise definition to implement against. The ambiguity in the spec is what allowed weak validation to be written in the first place.

So: static testing would not have found this bug directly, but it could have prevented the condition that caused it. Dynamic testing is what actually exposed it.

Assignment 2 — Test Plan: Task Submission Feature

Scope
In Scope
CHECKBOX task: mark complete, toggle back, verify state change
TEXT task: submit a written response, revise it, confirm revision replaces original
LINK task: submit a valid certificate URL, attempt to submit an invalid URL
Task state transitions: Not Started → In Progress → Submitted / Completed
Progress percentage update on the phase page immediately after submission
Progress update reflected on the dashboard after submission
Error states: empty TEXT submit (button disabled), invalid URL (inline error, blocked)
Persistence: submitted data survives a page refresh (spec success metric)
URL validation edge cases: near-valid strings like https://,  https:// alone, spaces-only
Out of Scope
Admin approving or rejecting submissions (non-goal in spec)
File upload as evidence (non-goal)
Email or in-app notifications on submission (non-goal)
Peer-visible or public submissions (non-goal)
Admin submitting on behalf of a member


Entry Criteria
The Task Submission feature is live and accessible at betaninjas-seo-learning-tracker.vercel.app
A test member account exists with access to Phase 1 and at least one task of each type (CHECKBOX, TEXT, LINK)
The phase page and dashboard load without JS errors before testing begins
The previous bug (https://, accepted as valid) has been confirmed as a known open defect before this test run


Exit Criteria
All 5 test cases executed and results recorded
TC-01, TC-02, TC-03, and TC-05 must pass for the feature to be considered releasable
TC-04 (the known URL validation bug) is expected to FAIL until fixed — result should be logged as a confirmed defect
No failed submission shows a success state (spec success metric)
Submitted data persists after page refresh for all task types
Progress percentage updates without a full page reload


Test Cases

TC-01: CHECKBOX Task — Mark Complete and Toggle Back (Happy Path)
Step 1
Log in and navigate to a phase containing a CHECKBOX task.
Step 2
Note the current progress percentage on the phase page.
Step 3
Open the task. Confirm it shows as Not Started or In Progress.
Step 4
Click the checkbox to mark it complete.
Step 5
Without refreshing, return to the phase page and note the new progress percentage.
Step 6
Open the task again and click the checkbox to uncheck it.
Step 7
Return to the phase page and confirm the percentage has decreased back.
Expected
Checking marks the task Completed and the phase progress percentage increases immediately (no reload). Unchecking returns the task to Not Started and the percentage decreases. One click each way as specified.
Label
Validation — Does one-click toggle feel safe for a destructive action? Is there any risk of accidental unchecking? Spec allows it — but a real user losing progress accidentally is a usability concern worth noting.


TC-02: TEXT Task — Submit and Revise a Response (Happy Path)
Step 1
Log in and open a TEXT task.
Step 2
Confirm the task state changes to In Progress on first open.
Step 3
Type a response of 2-3 sentences in the textarea.
Step 4
Click Submit.
Step 5
Confirm the task state shows Submitted and the phase progress increases.
Step 6
Refresh the page. Confirm the response is still visible.
Step 7
Edit the response text and click Submit again.
Step 8
Confirm the revised response has replaced the original (not appended).
Expected
Submission saves and is visible on return. Progress updates without reload. After revision, only the new response exists. Matches spec rule: revision replaces, does not append.
Label
Verification


TC-03: LINK Task — Submit a Valid Certificate URL (Happy Path)
Step 1
Log in and open a LINK task (e.g. Phase 1, Task 30).
Step 2
Paste a real, well-formed URL (e.g. https://academy.hubspot.com/certificate/sample).
Step 3
Confirm the field accepts the input without showing an error.
Step 4
Click Submit Link.
Step 5
Confirm 'Submitted!' appears and the task state updates to Submitted.
Step 6
Refresh the page and confirm the submitted URL is still visible in the field.
Step 7
Check the phase page — confirm progress percentage has increased.
Expected
Valid URL is accepted, submission saved, task moves to Submitted state, progress updates. Submitted URL persists after refresh.
Label
Verification


TC-04: LINK Task — Near-Valid Invalid URL Accepted (Known Bug — Expected FAIL)
Step 1
Log in and open a LINK task.
Step 2
In the URL field, type exactly: https://,
Step 3
Observe whether the field shows a green checkmark or a red error.
Step 4
Click Submit Link.
Step 5
Observe whether 'Submitted!' appears or whether an inline error blocks the submission.
Step 6
Check whether the task state has changed to Submitted.
Expected
EXPECTED (per spec): Field should show an inline error. Submit should be blocked. Task state should remain In Progress. ACTUAL (observed): Field shows green checkmark. 'Submitted!' appears. Task marked as Submitted. — THIS IS A CONFIRMED BUG.
Label
Verification — Verification — spec rule violated: 'a member cannot submit an invalid URL for a LINK task'


TC-05: Progress Percentage Accuracy Across Multiple Submissions (Edge Case)
Step 1
Log in on a phase with at least 3 tasks of mixed types, all Not Started.
Step 2
Note the starting progress percentage (should be 0%).
Step 3
Submit or complete one task. Note the new percentage.
Step 4
Submit a second task. Note the new percentage again.
Step 5
Complete all tasks in the phase. Confirm percentage shows 100%.
Step 6
Navigate to the dashboard. Confirm the overall progress bar also reflects the updated phase.
Step 7
Verify rounding: on a 3-task phase, 1 done = 33%, not 33.33%.
Expected
Progress percentage on phase page and dashboard updates after each submission without a page reload. Percentages round to the nearest whole number. Dashboard overall progress reflects phase changes in real time.
Label
Validation — If updates lag visually by more than ~1 second, flag as a UX concern even if technically correct.



Assignment 3 — Our Last Sprint: Honest Look
Sprint: Bubbles Playwright Automation — Channel Flows & Recording Options


When Did QA Actually Get Involved?
QA got involved after development was already complete  and in several cases after the feature had been live in production for some time. When I started writing Playwright tests for the Bubbles app, there was no spec to test against. The live app was the spec. That means any bugs I found during automation had already been visible to real users  we had simply not formally caught or documented them yet.

For the Channel flows (create, rename, delete) and the Recording Options flows, test design started purely in response mode: I opened the app, observed what it did, and wrote tests to match observed behaviour. This is the opposite of shift-left. QA was not in the room when requirements were written, not in planning when stories were estimated, and not consulted on what 'done' meant before development started.


One Real Example Each of QA, QC, and Testing
QA — Prevention
When writing the Channel rename/delete spec, I noticed early on that the styled-component class names Bubbles generates change on every build. Before writing a single assertion, I made the deliberate decision to use wildcard attribute selectors and placeholder-based locators instead of class names. This is quality assurance: I identified a structural risk before the test was written and the decision prevented an entire category of brittle failures from ever existing. The tests never broke because of a build-time class name change.
QC — Finding a Problem
While testing the auth flow, the Continue with Email button stayed disabled even after a valid email address was typed into the field. I reproduced it consistently: the button only enabled after the input lost focus (blur event). This was a React form validation timing issue the component state was not updating on each keystroke, only on blur. I found this through active exploratory testing, documented it with exact reproduction steps, and identified the fix: use .fill() + .blur() or pressSequentially() + .blur() in Playwright. That is QC: finding a real defect in an existing shipped feature.
Testing — Running a Test
Running the bubblesRecordingOptions.spec.ts suite against the live app is the clearest example of testing in the literal sense. The suite has structured test cases with defined steps and expected outcomes. Each run produces a pass or fail result against a specific assertion. The current known failure is the Delete bubble modal not opening before the assertion fires the test runs, it fails, it returns a specific failure message. The test is doing exactly what a test is supposed to do: providing a binary signal on whether the feature behaves as expected.


Which Agile Ceremonies Did QA Join — and the Impact

Ceremony
QA Presence and What Slipped
Sprint Planning
Absent. Stories were created and estimated without QA input. Test effort was not part of story points. Result: dev tasks closed as done with no test cases in place. Testing happened after the sprint or not at all.
Daily Standup
Partially present. QA progress was not a standing agenda item. The modal race condition blocker was not raised in standup — it was found independently during a solo test run.
Sprint Review
Absent. Features were demonstrated without a QA sign-off. There was no moment where tested vs untested was visible to stakeholders.
Retrospective
This assignment is the first structured retro the automation sprint has had. No formal retro occurred at the end of the last cycle.


The most impactful gap was sprint planning. Without QA at planning, test work has no time allocation, no definition of done, and no visibility as part of the sprint commitment. Automation gets treated as a parallel side track rather than part of delivery.


One Concrete Change for Next Sprint
QA Writes Acceptance Criteria for Every Story at Planning — Before Estimation
Before any story is pointed or committed, QA adds at least two acceptance criteria to the ticket: one happy path and one error or edge case. These sit in the ticket alongside the dev requirements and form part of the definition of done.

This is specific and doable today. It requires no new tools, no new process, just 10 minutes per story in planning and one standing question: 'QA, what would make this testable and done?'

Applied to today's bug: if the LINK task submission story had carried the criterion 'a URL consisting only of https:// followed by non-domain characters must be rejected with an inline error' the developer would have had a precise definition to build against, and the validation bug would have been caught at code level rather than discovered by opening the live app and typing into a field.


Submitted by Deepti Bhatta  June 16, 2026
