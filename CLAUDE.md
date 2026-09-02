# claude-kit

Personal Claude Code kit: agents, skills and cross-project methods, published as
a plugin and served by its own marketplace from this same repository.

This file is the contract for working *on* this repo. It is not shipped as part
of what the plugin exports.

## Why this repo exists

Configuration under `~/.claude/` lives on one machine and does not exist in a
cloud session. Anything that does not travel with a repository disappears.

This repo is the single source of truth for everything that must behave the same
across every project and every machine.

Adopting it takes two steps, not one. The committed `.claude/settings.json` (see
`project-template/`) *registers* the marketplace and marks the plugin as enabled —
but since Claude Code 2.1.195 a plugin from an external source is **not installed
by that file alone**, so it silently does not load until someone runs
`claude plugin install claude-kit@aniol` — once per machine locally, and once per
cloud environment from its **Setup script**, which runs before Claude Code launches
and is then snapshotted. A `SessionStart` hook cannot stand in for it: hooks run
after Claude Code has loaded its plugins.

So principle 5 holds per environment, not per repository: the committed file
travels, the install does not. Check `claude plugin list` before concluding that a
skill is broken.

This is also why the repository stays public (principle 4 is not optional, then):
the setup script cannot install the plugin from a private repository, and it fails
silently into a snapshot that every later session reuses.

## This repo follows its own rules

The rules the kit hands to other projects — `rules/how-we-work.md` — apply here too, most
of all here, since this is where they get argued about. This repo adopts them exactly as
any other project does: the plugin's `SessionStart` hook copies that file to
`.claude/rules/how-we-work.md`, which Claude Code loads every session. Edit the canonical
file, never the copy. The copy is refreshed at session start, so a change lands in the
next session, not this one.

## Design principles

1. **Content goes in skills; agents stay thin.** Skills follow an open spec that
   other agents also understand; the plugin format is Claude Code's. The value
   must be portable and the wrapper disposable. A subagent says who it is and
   which skill to delegate to — not much more.

   Practical consequence: an agent whose `tools:` frontmatter omits `Skill`
   cannot reach the skill holding its content. Include it, or drop `tools:`
   entirely and inherit.

2. **A subagent only when separate context is needed.** If the work is noisy —
   browser navigation, DOM, screenshots, test output — and would eat the main
   conversation's context, it is a subagent. Otherwise it is a skill.

3. **Nothing project-specific.** This is how things are done in general. What
   belongs to one project belongs in that project's `CLAUDE.md`. An instruction
   naming a client, a concrete URL or a concrete table is in the wrong place.

4. **No secrets, ever.** This repo is public. Invented test users are fine;
   nothing else is.

5. **Identical locally and in the cloud.** Nothing that depends on an absolute
   path from one machine, a tool installed only at home, or network access to
   domains a container will not have open.

6. **Short, and about one job.** Everything here consumes context. Five sharp
   lines beat forty exhaustive ones — cut words, not substance.

   Length is a smell, not the criterion. Past ~200 lines, look for what is
   padding and remove it; do not compress prose that earns its place just to
   land under a number. A skill gets **split** for a different reason: it covers
   two jobs *and* something would plausibly load the second one without the
   first. Until that second consumer exists, splitting only buys indirection and
   one more description competing to activate.

## Before adding anything

The first question is always: **is this genuinely cross-project, or does it
belong to one project?** Ask it out loud before writing a line. If the answer is
"one project", it goes in that project's `CLAUDE.md`, not here.

## Choosing the mechanism

Skills load when they are judged relevant, so they suit conditional guidance
("when you do X, do it this way"). They are the wrong tool for rules that must
apply *always*.

| When the guidance applies | Mechanism |
|---|---|
| A kind of work ("when writing tests") | Skill |
| Work that is noisy and needs its own context | Thin subagent delegating to a skill |
| A specific, detectable moment (about to commit, just edited a file) | Hook with a matcher — `PreToolUse` can also deny with a reason, `PostToolUse` runs after |
| Genuinely every session | `SessionStart` hook — its stdout is injected as context, paid once per session |
| Only when explicitly invoked | Slash command under `commands/` |

`UserPromptSubmit` injects on every single turn. It is the most guaranteed and
the most expensive; it works against principle 6. Avoid it.

Never write a skill with an inflated description hoping it triggers every time.
If a rule needs to always apply, say so and pick a hook instead.

## Skill descriptions

Skills compete to activate through their `description`. Each one must state
*when* it applies, not only what it does — otherwise adding skills makes the
existing ones worse. `skills/firebase-emulator/SKILL.md` is the reference.

## Layout

```
.claude-plugin/
  plugin.json         the plugin manifest
  marketplace.json    marketplace "aniol", serving this repo as one plugin
agents/               thin subagents
skills/               the content: one directory per skill, each with SKILL.md
rules/                always-on rules, copied into each project by the hook below
hooks/hooks.json      SessionStart: materialises rules/ into .claude/rules/
project-template/
  settings.json       drop into a project's .claude/ to enable the plugin
```

`commands/` and `scripts/` do not exist yet; add them when a rule actually calls for
one. Inside a plugin, reference bundled files through `${CLAUDE_PLUGIN_ROOT}` — never an
absolute path (principle 5).

## Checks

```bash
claude plugin validate .
```

It checks less than it looks like it does. It passed on a manifest that declared
`"hooks": "./hooks/hooks.json"`, which made every install fail to load with `Duplicate
hooks file detected` — `hooks/hooks.json` is discovered automatically and the `hooks`
field is only for files at other paths. The check that catches that class of mistake is
`claude plugin update` followed by `claude plugin list` on a real install.

**Bump `version` in `.claude-plugin/plugin.json` with every change worth shipping.**
`claude plugin update` and auto-update compare that number, not commits: leave it alone
and every existing install stays on the copy it first fetched, silently, however many
times the marketplace is refreshed.

A cloud session does not pick that up on its own, and opening a new one does not help.
The setup script runs **once per environment** and its result is snapshotted: every later
session starts from the version installed when that snapshot was taken. Measured here: a
snapshot from the 27th was still serving 0.1.0 on the 31st, with `main` at 0.2.0, in a
container whose clone of this repo was minutes old.

That is why the committed `.claude/settings.json` carries a `SessionStart` hook running
`claude plugin update` — for **both** scopes, since `enabledPlugins` produces a
*project*-scope install that the default `user` scope leaves alone. The hook travels with
the clone, which *is* fresh every session, and does not depend on the installed version.

Measured, twice: in an interactive session the hook does fix the session it runs in — at
startup, not mid-session: a release published while a session is open waits for the next. It
took the install from 0.2.0 to 0.2.1 in both scopes and the new skill descriptions were
live in that same session, despite the CLI printing "Restart to apply changes" — skills
are re-read, plugins are not reloaded. In a headless `claude -p` run the same hook updates
the disk and the session keeps serving the old skills.

So when a skill seems broken, check `claude plugin list` against
`.claude-plugin/plugin.json` before looking anywhere else.

Also check by hand: no secrets, no project names, no absolute paths, and no skill
that has quietly grown into two.
