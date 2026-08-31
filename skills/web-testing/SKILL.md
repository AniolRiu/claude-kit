---
name: web-testing
description: Verify that a web app actually works end-to-end by driving a real browser with Playwright against the LOCAL instance. Use after a refactor, which the build cannot validate — it compiles green with routes, payload keys, DOM ids and anything visual broken. Use when a user-facing feature has just been implemented or changed, when an auth or permission flow needs confirming, when a bug report needs reproducing through the UI, or when checking that a user journey has not regressed. Covers finding the documented local environment and refusing to guess it, writing throwaway scripts, selecting by role and label, waiting for state, classifying a failure as app bug / test bug / incomplete environment, and keeping durable flow knowledge. Not for performance benchmarking, load testing, or unit-level assertions. [marca-versio: 0.2.1]
---

# Web testing against a real local environment

Drive a real browser against the app to find out whether it actually works. This
is investigation, not test-suite authorship: write throwaway Playwright scripts,
run them, and report what is true.

## Why the build is not evidence

**A refactor is not finished until a browser has been through it.** The build does not
cover routes, the keys of a payload sent to a cloud function, DOM ids, or anything
visual: it compiles exactly as green with all of those broken.

Three broke at once in a single change — the keys of a shorthand object going to a
callable, a name leaking into the title bar, and a map going blank past a zoom level.
The build caught none of them and neither did reading the diff. All three were found by
someone clicking.

## 1. Local only. Always.

Test the **local instance** of the app. Never a staging URL, never a production
URL, never anything found in a README, a `.env`, a deploy config, or a commit
message that points at a real deployment.

Concretely, before writing a single line of test code:

1. Read the project's `CLAUDE.md` and find the section titled **"Test environment"**.
2. Take the base URL, the start command, the test users, and the reset procedure
   from that section.
3. If the app is not running yet, start it with the command that section gives you,
   and wait until it actually answers before continuing. In a cold cloud session
   that is **minutes, not seconds**: dependencies may need installing, and local
   backends often download a runtime on first start. Start it in the **background**
   and poll the base URL, rather than blocking on a foreground command that will
   hit a timeout. A start that looks stalled usually is not — read the log it is
   writing instead of killing it and retrying. If the section documents more than
   one command, start them in the order given and wait for each.

**If `CLAUDE.md` has no "Test environment" section, stop.** Do not guess
`http://localhost:3000`. Do not scan `package.json` for a dev script and hope.
Do not probe ports. Return immediately with a report that says: the project has no
documented test environment, this is what is missing (base URL, start command,
test credentials, reset procedure), and testing cannot proceed until someone
writes that section. Suggest the `firebase-emulator` skill if the project is on
Firestore.

A wrong URL is worse than no test: it either fails for reasons that have nothing
to do with the code, or — far worse — it writes to something real.

## 2. First, a browser that actually launches

Playwright is usually **not** a dependency of the project under test, and in a
container or cloud session the browsers are pre-installed but need not match the
version npm hands you. Settle both once, before writing any flow.

**Can a script outside the repo import `playwright`?** Resolution is relative to
the script, not to the repo, so the project's `node_modules` does not help.

```
mkdir -p /tmp/wt && cd /tmp/wt && node -e "require.resolve('playwright')" || echo MISSING
```

If it is missing, do not add it to the project's `package.json` — you never grow
the repo. Install it in that scratch directory and run the scripts from there, and
skip the browser download: the binaries are already on disk, and fetching them is
slow at best and blocked at worst.

```
npm init -y >/dev/null && PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm i playwright
```

**Never run `playwright install` or `npx playwright install`.** Managed
environments block it. Locate what is already there instead, and derive the path —
never hardcode a build number:

```
PW=${PLAYWRIGHT_BROWSERS_PATH:-$HOME/.cache/ms-playwright}; ls "$PW"
CHROME=$(ls -d "$PW"/chromium 2>/dev/null || ls -d "$PW"/chromium-*/chrome-linux/chrome 2>/dev/null | head -1)
```

**Then pass `executablePath` explicitly**, always. Do not rely on Playwright
finding the browser itself: it looks for the build number *its* version expects,
which is often not the one on disk, and the error it prints tells you to run the
one command you must not run.

If a browser still will not launch, **stop and report an incomplete environment**,
exactly as for a missing "Test environment" section. Do not fall back to `curl`,
do not assert against HTML fetched without a browser, and never report a flow as
passing that you did not actually drive.

## 3. How to work

Write the Playwright script next to that scratch `node_modules` and run it with
Bash. Do not add test files to the repository; the repo's own test suite is not
your concern and must not be grown silently.

```
cat > /tmp/wt/checkout.mjs <<'SCRIPT'
import { chromium, expect } from 'playwright/test';

const browser = await chromium.launch({ executablePath: process.env.WT_CHROME });
// ... one flow, start to finish
SCRIPT

WT_CHROME="$CHROME" node /tmp/wt/checkout.mjs
```

Run headless by default. Keep each script focused on one flow, so that when it
fails you know what failed.

## 4. Selectors

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

## 5. Waiting

Wait for **state**, never for time. `page.waitForTimeout(2000)` is banned — it is
how a suite becomes slow and flaky at the same time.

Wait for the thing you actually care about:

- `await expect(page.getByRole('alert')).toHaveText('Saved')`
- `await expect(page.getByRole('row')).toHaveCount(3)`
- `await page.waitForURL('**/orders/*')`
- `await expect(locator).toBeVisible()`

If something is genuinely asynchronous and unobservable in the DOM, that is a gap
in the UI — the user cannot tell either. Say so.

## 6. When a test fails

A failure is information, and it must be classified correctly before anyone
touches code. Every failure is exactly one of three things:

**A bug in the app.** The app is reachable, the flow is correct, the selectors
resolve, and the behaviour is wrong. Report what was done, what was expected, what
happened, and the smallest reproduction.

**A bug in the test.** Wrong selector, wrong assumption about the flow, a race
introduced by the script, a test user that lacks the role the flow requires. Fix
the script and run again. Own it plainly in the report — a false alarm costs
someone an afternoon.

**An incomplete environment.** No browser, the emulator not running, seed data
without the record the flow needs, an unset environment variable, a service the
page calls that is down. This is not an app bug and must never be reported as one.
Say exactly what is missing and how to provide it.

If you cannot tell which of the three it is, say so and say what evidence would
settle it. "I don't know yet" is a legitimate report; a confident wrong diagnosis
is not.

**Never weaken an assertion to make a test pass.** Not by loosening the expected
text, not by broadening a selector until it matches something, not by dropping a
count check, not by wrapping it in a try/catch, not by retrying around a real
failure. A green test that asserts nothing is worse than a red one, because it
will be trusted. If an assertion is genuinely wrong about the intended behaviour,
change it deliberately and say in the report what changed, and why.

## 7. Durable knowledge: `.claude/webtester/FLOWS.md`

Maintain one file: `.claude/webtester/FLOWS.md`, at the root of the project being
tested. It holds what is still true next week:

- how to log in, and which test user has which role
- what the main flows are, and the sequence of steps each one takes
- selectors that turned out to be fragile, and what to use instead
- app behaviours that reliably surprise a newcomer (a redirect, a debounce, a
  modal that must be dismissed before the page is interactive)
- flows that are known broken, with a date

Write it for the next agent, who has never seen this app. Update it when something
durable was learned and leave it alone otherwise. **Keep it under 200 lines**: when
it gets close, consolidate — merge overlapping flows, delete notes about code that
no longer exists, collapse three observations into the rule they were pointing at.
Never let it become an append-only log.

**Execution logs live in `/tmp` and are meant to be lost.** Script output, traces,
screenshots, HTML dumps, timings — none of it goes in `FLOWS.md` or anywhere else
in the repository. It is evidence for this run's report, not knowledge.

### What must never go in that file

`FLOWS.md` gets committed. Treat it as public.

- **No cookies, session tokens, auth headers or API keys.** Not truncated, not
  expired, not as an example. If one is needed to debug, keep it in `/tmp` for that
  run and let it disappear.
- **No screenshots of authenticated pages** — they carry real names, emails,
  balances and record IDs — and no real user data copied out of the app.
- Test credentials from the documented "Test environment" section are fine: they
  are already public in `CLAUDE.md` and only exist against the local emulator.

## 8. The report

Return plain text, not a file:

- **What was tested** — the flow, the base URL, the user and role.
- **Result** — pass or fail, per assertion that matters.
- **On failure** — the classification from §6, the evidence, the minimal repro.
- **UX observations** — anything that worked but felt wrong: an unlabelled control,
  a silent failure, a state with no feedback, a dead end. Keep this separate from
  the functional results.
- **Next action** — the single most useful thing to do next, and who should do it.
