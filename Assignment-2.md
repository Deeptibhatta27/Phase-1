# Software Testing Assignment
### BetaninjasSEOLMS — Login Feature
*Black Box · Grey Box · White Box Testing*

---

## Step 1 — Black Box Testing

Tested purely as a user — no knowledge of code or architecture.

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| TC-01 | Wrong Password | Enter valid email + incorrect password → click Login | Error message shown, user stays on login page |
| TC-02 | Empty Fields | Leave email and password blank → click Login | Validation errors shown for both fields, no redirect |
| TC-03 | Valid Login | Enter correct email + correct password → click Login | User redirected to dashboard / home page |
| TC-04 | Invalid Email Format | Enter `notanemail` as email + any password → click Login | Inline error: "Enter a valid email address" |
| TC-05 | Very Long Input | Enter 500-char string in email + password fields → click Login | App handles gracefully — no crash, shows error or truncates |

---

## Step 2 — Grey Box Testing

After reading `auth.ts` and `middleware.ts`, two test cases were identified that would not be obvious from the UI alone.

| ID | Test Case | Steps | Expected Result & Code Insight |
|----|-----------|-------|-------------------------------|
| GBC-01 | Non-admin user accessing `/admin` route | Log in as a non-admin user (role = `member`). Manually navigate to `/admin` in the browser. | **Insight from `middleware.ts`:** `isAdminRoute` check redirects non-admin users to `/dashboard`. User should be redirected away from `/admin` without seeing any admin content. |
| GBC-02 | Already logged-in user revisiting `/login` | Log in successfully. Open a new tab and manually navigate to `/login`. | **Insight from `middleware.ts`:** `isLoggedIn && isLoginPage` triggers redirect to `/`. User should be automatically redirected to the home page, not shown the login form again. |

---

## Step 3 — White Box Testing: `getDaysRemaining`

### Function Signature & Behaviour

```ts
export function getDaysRemaining(dueDate: Date, now: Date = new Date()): number {
  const diff = dueDate.getTime() - now.getTime()
  return Math.ceil(diff / (1000 * 60 * 60 * 24))
}
```

| Property | Detail |
|----------|--------|
| **Inputs** | `dueDate` — the deadline (Date object). `now` — current time, defaults to `new Date()` if not provided. |
| **Returns** | A number representing days remaining. Positive = future, `0` = due right now, Negative = overdue. |
| **Logic** | Subtracts `now` from `dueDate` in milliseconds, divides by ms-per-day, applies `Math.ceil` to round up. |

### Edge Cases

| ID | Edge Case | Input | Analysis |
|----|-----------|-------|----------|
| WB-01 | Due date is today (same day, slightly later time) | `dueDate = today at 23:59`, `now = today at 08:00` | `diff` is positive but less than 86,400,000ms. `Math.ceil(0.67) = 1`. Returns `1`, meaning "due today". Correct and expected. |
| WB-02 | Due date is in the past (overdue) | `dueDate = yesterday`, `now = today` | `diff` is negative (e.g., `-86,400,000ms`). `Math.ceil(-1.0) = -1`. Returns a negative number. The caller must check for negatives to detect overdue state — the function itself gives no error. |
| WB-03 | Due date is exactly now (same millisecond) | `dueDate === now` (identical timestamp) | `diff = 0`. `Math.ceil(0) = 0`. Returns `0`. This means "due right now" — any caller treating `0` as "today" or "completed" must handle this carefully. |

---

## Step 4 — Testing Types for the Login Feature

| Testing Type | Why It Matters for Login |
|--------------|--------------------------|
| Functional Testing | Login is a core user action — functional tests verify that correct credentials grant access and incorrect ones are rejected, ensuring the feature works as designed. |
| Security Testing | Login handles credentials and authentication tokens, making it a prime target for attacks like brute force, credential stuffing, and session hijacking — security testing ensures these are mitigated. |
| Regression Testing | Any code change (new feature, dependency update) could accidentally break the login flow, so regression tests catch unintended breakage before it reaches users. |
| Accessibility Testing | Login pages must be usable by people with disabilities — screen readers, keyboard navigation, and color contrast all matter to ensure everyone can authenticate. |
| Performance Testing | If the login endpoint is slow under load, users experience timeouts or delays — performance tests ensure the auth system holds up during peak traffic. |

---

## Step 5 — Reflection

Grey box testing found the most interesting issues. Reading `auth.ts` and `middleware.ts` revealed behaviours — like the automatic redirect for already-logged-in users and the role-based admin guard — that would never surface from just clicking around the UI. Black box testing is easy to start but stays shallow; you only find what a typical user might bump into. White box testing on `getDaysRemaining` was the hardest, because you have to reason through the math carefully without running the code — figuring out what `Math.ceil` does to `0` or a negative number requires real attention. Black box was the easiest but the least revealing; grey box hit the sweet spot between effort and insight for a feature like login that has real security logic underneath.
