# claude-kit

A personal kit for [Claude Code](https://claude.com/claude-code): the agents,
skills and cross-project methods I reuse everywhere, so that they exist in a
cloud session as well as on my own machine. What is here today is built around a
single idea — **test against a real local environment, never against something
deployed.**

This repository is both the plugin and the marketplace that serves it. The rules
for what belongs here are in [`CLAUDE.md`](CLAUDE.md); the short version is that
the content lives in skills, the agents stay thin, and nothing project-specific
gets in.

## Install

```bash
claude plugin marketplace add AniolRiu/claude-kit
claude plugin install claude-kit@aniol
```

Or, to enable it automatically in a project, copy
[`project-template/settings.json`](project-template/settings.json) into that
project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "aniol": {
      "source": { "source": "github", "repo": "AniolRiu/claude-kit" }
    }
  },
  "enabledPlugins": {
    "claude-kit@aniol": true
  }
}
```

## What's inside

### `web-testing` (skill) and `webtester` (agent)

Drives a real browser with Playwright against the **local** instance of the app,
to find out whether a feature actually works end to end.

The method is the skill; the agent is a thin wrapper around it, and exists only
because a browser session is noisy enough to be worth keeping out of the main
conversation's context. The skill works on its own too, without the agent.

- Reads the **"Test environment"** section of the project's `CLAUDE.md` for the
  base URL, start command, test users and reset procedure. If that section is
  missing it **stops and says so** rather than guessing a URL — a wrong URL either
  fails for unrelated reasons or writes to something real.
- Writes throwaway scripts to `/tmp` and runs them with Bash; it does not grow the
  repo's test suite behind your back.
- Selects by role and label (`getByRole`, `getByLabel`), never by structural CSS.
- Waits for state, never for time.
- On failure, classifies it as **app bug / test bug / incomplete environment**, and
  never weakens an assertion to turn a test green.
- Maintains `.claude/webtester/FLOWS.md` with durable knowledge (how login works,
  what the flows are, which selectors are fragile), kept under 200 lines and
  consolidated as it grows. Run logs stay in `/tmp` and are meant to be lost, and
  no cookies, tokens or screenshots of authenticated pages ever go in that file —
  it gets committed.

### `uix-expert` (agent)

UI/UX review and design. Currently a **placeholder** with correct frontmatter and a
short generic body, waiting for its real prompt.

### `firebase-emulator` (skill)

Turns a Firestore project into something you can actually test against. It
activates on its own when a Firebase project has no working local environment —
in particular when `CLAUDE.md` has no "Test environment" section.

It covers the `emulators` block in `firebase.json` (Firestore, Auth, UI, fixed
ports, `singleProjectMode`, real rules and indexes); connecting the app via
`FIRESTORE_EMULATOR_HOST` / `FIREBASE_AUTH_EMULATOR_HOST` and
`connectFirestoreEmulator` / `connectAuthEmulator`; seeding `qa@test.local` plus
one user per role and freezing it with `firebase emulators:export ./seed`;
getting realistic data through an anonymised snapshot produced on a trusted
machine, **never by pointing at production**; writing the `CLAUDE.md`
"Test environment" section that `webtester` depends on; and verifying the whole
thing really starts before calling it done.

It closes with what the emulator will *not* tell you — composite indexes, volume,
latency and everything in front of the app — all of which must be checked against
the actual deployment.

## Layout

```
CLAUDE.md             why this repo exists and what may go in it
.claude-plugin/
  plugin.json         claude-kit, v0.1.0
  marketplace.json    marketplace "aniol", one plugin at "./"
agents/
  webtester.md        identity, and delegation to skills/web-testing
  uix-expert.md
skills/
  web-testing/SKILL.md
  firebase-emulator/SKILL.md
project-template/
  settings.json       drop into a project's .claude/ to enable the plugin
```

## Development

```bash
claude plugin validate .
```
