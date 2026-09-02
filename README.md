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

Three pieces, and the kit only behaves as intended with all three:

| Piece | Where it lives | Paid |
|---|---|---|
| The install itself | `~/.claude/` of a machine or a container | once per machine, once per cloud environment |
| The project opting in | `.claude/settings.json`, committed | once per project |
| Staying current | a `SessionStart` hook in that same file | free, every session |

### 1. The install

On your own machine:

```bash
claude plugin marketplace add AniolRiu/claude-kit
claude plugin install claude-kit@aniol
```

Once per machine, not once per project: it lives in `~/.claude/` and stays there.

For a **cloud environment** that same `~/.claude/` belongs to the container, and no
committed file can create it. It goes in the **Setup script** field of the environment
dialog at claude.ai/code:

```bash
#!/bin/bash
# bump this number to force the snapshot to rebuild: 1
{
  claude plugin marketplace add AniolRiu/claude-kit
  claude plugin install claude-kit@aniol
  claude plugin update claude-kit@aniol
  claude plugin list
} > /var/log/claude-kit-setup.log 2>&1 || true
```

The `|| true` matters: a setup script that exits non-zero stops the session starting.
Keep the log — it is the only record of what the environment was built with, and it
survives into every session made from that snapshot.

That script runs as root before Claude Code launches, and Anthropic snapshots the
filesystem afterwards. **It is paid once per environment and the result is frozen**:
every later session, new or resumed, starts with the version installed the day the
snapshot was taken, however far `main` has moved on. Measured: a snapshot from the 27th
was still serving 0.1.0 on the 31st with `main` at 0.2.0, in a container whose clone of
this repo was minutes old. What forces a rebuild is the script's **text** changing —
hence the number to bump in the comment.

### 2. The project opts in

Copy [`project-template/settings.json`](project-template/settings.json) into the
project's `.claude/settings.json` and commit it. It registers the marketplace, enables
the plugin, and carries the hook from step 3.

**The `enabledPlugins` part is not enough on its own.** Since Claude Code 2.1.195 it
registers the marketplace and marks the plugin as enabled, but a plugin from an external
source is not installed by settings alone — it does not load until someone has run the
install from step 1 on that machine or in that environment. Nothing warns you in
conversation: the skills are simply absent, and Claude answers as if the kit did not
exist. `claude plugin list` is the check.

That is the whole of it: there is nothing to paste. The plugin carries its own
`SessionStart` hook, which writes `.claude/rules/how-we-work.md` into the project at
startup; Claude Code reads memory files before hooks run, so the rules it writes are in
context from the following session on. Commit the file — it is generated and must never
be edited by hand, but having it in the repo means the rules still hold in a clone where
nobody has run step 1.

That also means the two `SessionStart` hooks are one behind each other. Plugins load
before hooks run, so the plugin hook that fires in a session is the one from the version
installed when it started — not the one step 3 has just fetched. Going from a version
without the hook to one with it therefore takes two sessions: the first updates the
plugin, the second writes the rules.

### 3. Staying current

The `SessionStart` hook in that same file is what keeps a session on the published
version instead of the installed one:

```json
"hooks": {
  "SessionStart": [
    {
      "matcher": "startup|resume",
      "hooks": [
        {
          "type": "command",
          "command": "claude plugin marketplace update aniol >/dev/null 2>&1; { claude plugin update claude-kit@aniol; claude plugin update claude-kit@aniol --scope project; } 2>&1 | grep -o 'updated from .*' || true",
          "timeout": 90
        }
      ]
    }
  ]
}
```

It lives in the repository and **not inside the plugin** on purpose: it has to work when
the installed copy is old, and an old copy would not carry it. The clone *is* fresh every
session; the snapshot is not.

It updates **both scopes**. `enabledPlugins` produces a *project*-scope install and
`claude plugin update` touches only `user` by default, so with a single line the install
that actually loads is the one left behind.

Measured three times, in two repositories: at session start the hook takes both scopes
from the snapshot's version up to the published one, and the new skills are live **in
that same session**, despite the CLI printing `Restart to apply changes` — skills are
re-read even though plugins are not reloaded. Two limits worth knowing:

- A headless `claude -p` run updates the disk but keeps serving the old skills.
- A version published while a session is open waits for the next session.

So **bumping `version` in `.claude-plugin/plugin.json` is what ships a change**, and the
setup script only needs touching again if you want the snapshot itself rebuilt.

On your own machine there is no snapshot, so the hook is enough by itself. Auto-update is
worth turning on anyway — `/plugin` → **Marketplaces** → `aniol` → **Enable auto-update**
— because third-party marketplaces have it **off** by default;
`claude plugin marketplace update aniol` refreshes it by hand.

### The repository has to stay public

Installing from the setup script was tested against a private repository and failed on
the clone:

```
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

Inside a running cloud session the proxy injects git credentials, so a private
marketplace clones fine there. At setup-script time those credentials do not exist yet —
git got far enough to ask for a username. It is authentication, not protocol: the clone
was already going over HTTPS, so `CLAUDE_CODE_PLUGIN_PREFER_HTTPS` makes no difference.

The failure is silent: `|| true` keeps the session starting, the snapshot is then taken
**without** the plugin, and every later session starts broken too. Recovering means
changing the script's text, which is what forces the rebuild — making the repository
public again does nothing on its own.

Keeping it private would mean putting a token in the environment's variables and
configuring a git credential helper in the setup script. Those variables are visible to
anyone using the environment, and nothing here is worth that.

### When something does not work

```bash
claude plugin list                  # installed, enabled, and at which version in each scope
claude plugin details claude-kit    # which skills and agents it contributes
cat /var/log/claude-kit-setup.log   # cloud only: what the environment was built with
```

Compare the version against `.claude-plugin/plugin.json` on `main` before looking
anywhere else. Opening a new cloud session does **not** fix an old version — it is a new
container from the same snapshot.

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
- Verifies the documented environment instead of trusting it: requests every port the
  section lists, never believes a tool's own "ready" banner, and tries the other loopback
  name before declaring an app dead.
- On failure, classifies it as **app bug / test bug / incomplete environment / wrong
  documentation**, and never weakens an assertion to turn a test green. It reports a
  false line in `CLAUDE.md` rather than editing it, and never writes the section it
  found missing — a fabricated test environment turns the guard inside out.
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

### `commit` (skill)

How to consolidate work once asked: read the diff before writing the message, clean what
should not ship, check it builds, split by reason for change, and propose every message —
all of them at once, with their files — before running git. Deciding *whether* to commit
is not in it: that is the user's, and that rule lives in `rules/how-we-work.md` instead,
because a skill only loads when it is judged relevant and that rule has to hold always.

### `rules/how-we-work.md`

The rules that have to hold in every session, whatever the conversation is about: the
language split, never committing unasked, keeping options open, saying only what changes
something, being critical rather than agreeable. They are not a skill on purpose — a
skill loads only when it is judged relevant, and these have to be there before anyone
knows the session will need them.

A plugin cannot ship a `CLAUDE.md`; it can ship a hook. So the canonical text lives here
and the plugin's `SessionStart` hook copies it into each project as
`.claude/rules/how-we-work.md`, which Claude Code loads every session at the same
priority as `.claude/CLAUDE.md`. Edit the file in the plugin and every project picks the
change up on its next session — the same path as any other change to the kit, and the
reason the rules are no longer a fragment to paste by hand.

### `firestore-rules` (skill)

Writing Firestore and Storage rules and proving them against the emulator. The premise is
that the client is not a security boundary — a hidden button is not an authorization
check — and the weight of the skill is on the negative cases: a suite that only tests what
should work is passed perfectly by rules that allow everything.

### `project-plan` (skill)

Keeps one document per project of what is pending and what is still undecided —
adopting whatever planning file the project already has, or creating `.claude/PLAN.md`.
Entries are one line; a decision is recorded with its reason at the moment it is
taken; pending work and open questions are state and get deleted rather than
archived, while decisions are kept. This repo's own is in
[`.claude/PLAN.md`](.claude/PLAN.md).

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
  plugin.json         claude-kit, v0.4.0
  marketplace.json    marketplace "aniol", one plugin at "./"
agents/
  webtester.md        identity, and delegation to skills/web-testing
  uix-expert.md       identity, and delegation to skills/ui-ux
skills/
  web-testing/SKILL.md
  ui-ux/SKILL.md
  project-plan/SKILL.md
  commit/SKILL.md
  firebase-emulator/SKILL.md
  firestore-rules/SKILL.md
rules/
  how-we-work.md      the always-on rules, canonical copy
hooks/
  hooks.json          SessionStart: writes rules/ into a project's .claude/rules/
project-template/
  settings.json       drop into a project's .claude/ to enable the plugin
  README.md           and the install step that settings alone does not do
```

## Development

```bash
claude plugin validate .
```

Bump `version` in `.claude-plugin/plugin.json` when shipping a change: updates compare
that number, not commits, and an unchanged version means no existing install ever sees
the change.
