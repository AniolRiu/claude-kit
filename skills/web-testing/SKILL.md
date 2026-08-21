---
name: web-testing
description: Verify that a web app actually works end-to-end by driving a real browser with Playwright against the LOCAL instance. Use when a user-facing feature has just been implemented or changed, when an auth or permission flow needs confirming, when a bug report needs reproducing through the UI, or when checking that a user journey has not regressed. Covers finding the documented local environment and refusing to guess it, writing throwaway scripts, selecting by role and label, waiting for state, classifying a failure as app bug / test bug / incomplete environment, and keeping durable flow knowledge. Not for performance benchmarking, load testing, or unit-level assertions.
---

# Web testing against a real local environment

Drive a real browser against the app to find out whether it actually works. This
is investigation, not test-suite authorship: write throwaway Playwright scripts,
run them, and report what is true.

## 1. Local only. Always.

Test the **local instance** of the app. Never a staging URL, never a production
URL, never anything found in a README, a `.env`, a deploy config, or a commit
message that points at a real deployment.

Concretely, before writing a single line of test code:

1. Read the project's `CLAUDE.md` and find the section titled **"Test environment"**.
2. Take the base URL, the start command, the test users, and the reset procedure
   from that section.
3. If the app is not running yet, start it with the command that section gives you,
   and wait until it actually answers before continuing.

**If `CLAUDE.md` has no "Test environment" section, stop.** Do not guess
`http://localhost:3000`. Do not scan `package.json` for a dev script and hope.
Do not probe ports. Return immediately with a report that says: the project has no
documented test environment, this is what is missing (base URL, start command,
test credentials, reset procedure), and testing cannot proceed until someone
writes that section. Suggest the `firebase-emulator` skill if the project is on
Firestore.

A wrong URL is worse than no test: it either fails for reasons that have nothing
to do with the code, or — far worse — it writes to something real.

## 2. How to work

Write the Playwright script to `/tmp` and run it with Bash. Do not add test files
to the repository; the repo's own test suite is not your concern and must not be
grown silently.

```
cat > /tmp/wt-checkout.mjs <<'SCRIPT'
import { chromium, expect } from 'playwright/test';
// ... one flow, start to finish
SCRIPT

node /tmp/wt-checkout.mjs
```

Run headless by default. Keep each script focused on one flow, so that when it
fails you know what failed.

## 3. Selectors

Use selectors that describe what the user sees, not how the page is built:

- `getByRole('button', { name: 'Save changes' })`
- `getByLabel('Email address')`
- `getByRole('heading', { name: 'Your orders' })`
- `getByText('Order confirmed')`

Never `.css-1x9vk2`, never `div > div:nth-child(3) > span`, never a class name that
a refactor or a utility-CSS change will invalidate. If an element genuinely cannot
be reached by role or label, that is an accessibility finding — report it, and use
a `data-testid` as the fallback, noting in the report that the element needs a
proper accessible name.

## 4. Waiting

Wait for **state**, never for time. `page.waitForTimeout(2000)` is banned — it is
how a suite becomes slow and flaky at the same time.

Wait for the thing you actually care about:

- `await expect(page.getByRole('alert')).toHaveText('Saved')`
- `await expect(page.getByRole('row')).toHaveCount(3)`
- `await page.waitForURL('**/orders/*')`
- `await expect(locator).toBeVisible()`

If something is genuinely asynchronous and unobservable in the DOM, that is a gap
in the UI — the user cannot tell either. Say so.

## 5. When a test fails

A failure is information, and it must be classified correctly before anyone
touches code. Every failure is exactly one of three things:

**A bug in the app.** The app is reachable, the flow is correct, the selectors
resolve, and the behaviour is wrong. Report what was done, what was expected, what
happened, and the smallest reproduction.

**A bug in the test.** Wrong selector, wrong assumption about the flow, a race
introduced by the script, a test user that lacks the role the flow requires. Fix
the script and run again. Own it plainly in the report — a false alarm costs
someone an afternoon.

**An incomplete environment.** The emulator is not running, the seed data does not
contain the record the flow needs, a required environment variable is unset, a
service the page calls is not up. This is not an app bug and must never be reported
as one. Say exactly what is missing and how to provide it.

If you cannot tell which of the three it is, say that too, and say what evidence
would settle it. "I don't know yet" is a legitimate report; a confident wrong
diagnosis is not.

**Never weaken an assertion to make a test pass.** Not by loosening the expected
text, not by broadening a selector until it matches something, not by dropping a
count check, not by wrapping it in a try/catch, not by retrying around a real
failure. A green test that asserts nothing is worse than a red one, because it
will be trusted. If an assertion is genuinely wrong about the intended behaviour,
change it deliberately and state in the report that what was being asserted
changed, and why.

## 6. Durable knowledge: `.claude/webtester/FLOWS.md`

Maintain one file: `.claude/webtester/FLOWS.md`, at the root of the project being
tested. It holds what is still true next week:

- how to log in, and which test user has which role
- what the main flows are, and the sequence of steps each one takes
- selectors that turned out to be fragile, and what to use instead
- app behaviours that reliably surprise a newcomer (a redirect, a debounce, a
  modal that must be dismissed before the page is interactive)
- flows that are known broken, with a date

Rules for this file:

- **Keep it under 200 lines.** When it gets close, consolidate: merge overlapping
  flow descriptions, delete notes about code that no longer exists, collapse three
  specific observations into the one general rule they were all pointing at. Do not
  let it become an append-only log.
- Write it for the next agent, who has never seen this app.
- Update it when something durable was learned; leave it alone otherwise.

**Execution logs live in `/tmp` and are meant to be lost.** Script output, traces,
screenshots, HTML dumps, timings — none of it goes in `FLOWS.md` or anywhere else
in the repository. It is evidence for this run's report, not knowledge.

### What must never go in that file

`FLOWS.md` gets committed. Treat it as public.

- **No cookies. No session tokens. No auth headers. No API keys.** Not even
  truncated, not even expired, not even as an example.
- **No screenshots of authenticated pages** — they carry real names, emails,
  balances, and record IDs.
- **No real user data** copied out of the app.
- Test credentials from the documented "Test environment" section are fine: they
  are already public in `CLAUDE.md` and only exist against the local emulator.

If a token is needed to debug, keep it in `/tmp` for that run and let it disappear.

## 7. The report

Return plain text, not a file:

- **What was tested** — the flow, the base URL, the user and role.
- **Result** — pass or fail, per assertion that matters.
- **On failure** — the classification from §5, the evidence, the minimal repro.
- **UX observations** — anything that worked but felt wrong: an unlabelled control,
  a silent failure, a state with no feedback, a dead end. Keep this separate from
  the functional results.
- **Next action** — the single most useful thing to do next, and who should do it.
