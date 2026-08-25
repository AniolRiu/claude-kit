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

To make a project ask for it, copy
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

**That file is not enough on its own.** Since Claude Code 2.1.195 it registers the
marketplace and marks the plugin as enabled, but a plugin from an external source
is not installed by settings alone — it does not load until someone runs
`claude plugin install claude-kit@aniol` on that machine. Nothing warns you in
conversation: the skills are simply absent, and Claude answers as if the kit did
not exist.

For **cloud sessions** the install lives in the container's `~/.claude/`, which no
committed file can create. It goes in the **Setup script** field of the cloud
environment (the environment dialog at claude.ai/code):

```bash
claude plugin marketplace add AniolRiu/claude-kit || true
claude plugin install claude-kit@aniol || true
```

That script runs as root before Claude Code launches, and Anthropic snapshots the
filesystem afterwards — so this is paid **once per environment**, not once per
session, and every later session starts with the kit already installed. The
`|| true` matters: a setup script that exits non-zero stops the session starting.

A `SessionStart` hook does **not** work for this, however tempting: it runs *after*
Claude Code launches, so a plugin installed there is not loaded in that session.

When something from the kit does not seem to work, check that first:

```bash
claude plugin list      # is claude-kit@aniol installed and enabled?
claude plugin details claude-kit   # which skills and agents it contributes
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

### `ui-ux` (skill) and `uix-expert` (agent)

Reviews an interface as the person using it: hierarchy, states, copy, spacing and
accessibility. Same split as above — the method is the skill, the agent is thin.

It exists as a separate agent because reading enough UI to have an opinion fills a
context window, and because the design mode of looking loses every argument it has
to share with the implementation mode, where "it already works" always wins.

- Looks at what the screen asks of the person before reading the files that build
  it, and treats the **states** — loading, empty, error, no-permission — as the
  place products usually fall apart, because seeded data hides them.
- Returns **three to five findings, ranked**, never a list of thirty nits. Each one
  says what is there, what problem it causes the person using it, and the concrete
  alternative — never the principle on its own.
- States its assumptions about who the user is, and separates what changes
  behaviour from what changes appearance.
- Is **read-only**: it does not edit the code and cannot run the app. When a state
  can only be judged by seeing it, it asks for a screenshot or for `webtester`.

### `plugin-smoke-test` (skill) — temporary

A canary. Asking `vols arròs?` should get back `pikachu` and nothing else, which
proves the plugin reached the project and that its skills trigger on their own.
Delete it once that has been confirmed.

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
  uix-expert.md       identity, and delegation to skills/ui-ux
skills/
  web-testing/SKILL.md
  ui-ux/SKILL.md
  firebase-emulator/SKILL.md
  plugin-smoke-test/SKILL.md   temporary canary, delete after checking
project-template/
  settings.json       drop into a project's .claude/ to enable the plugin
  README.md           and the install step that settings alone does not do
```

## Development

```bash
claude plugin validate .
```
