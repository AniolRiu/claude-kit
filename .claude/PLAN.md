# Plan

Ordered by impact: this is a flat set of independent work, not a phased project.

## Pending

- Check that `.claude/rules/how-we-work.md` survives a `/compact`. The documentation
  promises re-injection for the root `CLAUDE.md` and reloading for rules with `paths:`,
  and says nothing about rules without them. If it does not survive, these rules belong in
  the project's `CLAUDE.md` and the hook has to write a delimited section instead of a
  file. Checked with `/context` after a compact.
- Decide whether `uix-expert` writes its findings into the project it reviews.
  Blocked on the open question below; nothing else waits on it.
- Reconsider the `PreToolUse` hook for commits. It was ruled out on the grounds that the
  skill already requires proposing messages first; the same day, the agent broke that
  rule twice within minutes of writing it, because a skill only reaches a session when it
  is judged relevant and had not been loaded at all. Blocks nothing, but it is the one
  rule whose failure is written into history.
- Check whether `firebase-emulator`'s rules section should shrink to a pointer now that
  `firestore-rules` exists. The two have overlapped since 2026-08-29 and the emulator
  skill is the longest in the kit at 261 lines.
- Watch the kit's always-on cost. It was ~1,000 tokens with three skills; there are now
  six. If it passes ~2,000 per session, shorten descriptions without losing the *when*.
  The `project-template` fragment is the other half of the bill and grew by a third on
  2026-08-29; measured together they were 2.8k that day. Descriptions are not where the
  next cut necessarily is.
- Consider a `SessionStart` hook that prints the project's plan when one exists.
  Without it, this skill's "keep it updated" depends on the model noticing.

## Open questions

- **Should `uix-expert` persist anything in the reviewed project?** `webtester`
  keeps `.claude/webtester/FLOWS.md`; `uix-expert` is read-only and returns text.
  Blocks nothing today, but design conventions a project has already settled get
  re-derived every review. Settled by trying one review and seeing what was worth
  keeping.
- **Where do agents write, as a convention?** `.claude/<agent>/`, one file, under
  200 lines, committed and therefore public — this is stated only inside
  `web-testing`, as if it were particular to it. Settled once a second agent writes.

## Decided

- **2026-09-02 — The always-on rules ship inside the plugin, not as a fragment to paste.**
  `rules/how-we-work.md` is the canonical text, and the plugin's `SessionStart` hook copies
  it into each project as `.claude/rules/how-we-work.md`, which loads every session at the
  same priority as `.claude/CLAUDE.md`. A plugin cannot contribute a `CLAUDE.md` — the
  documentation is explicit — but it can contribute hooks. Replaces
  `project-template/CLAUDE.md`, whose every edit silently left adopters on the old text;
  the Catalan/English rule added the same day is what exposed it. Rules out `@` importing
  the clone: relative import paths resolve against the importing file and no cloud
  container has a clone to point at. The generated copy is committed, so the rules still
  hold where the install is missing.
- **2026-08-27 — The repository stays public.** A cloud environment's setup script
  has no git credentials, so it cannot clone a private marketplace; the install
  fails silently into a snapshot every later session reuses. Rules out keeping
  anything here that could not be public.
- **2026-08-27 — Content lives in skills; agents stay thin.** Skills follow an open
  spec, the plugin format is Claude Code's. Rules out agents that hold their own
  method.
- **2026-08-27 — `uix-expert` is read-only.** It reviews; the change belongs to
  whoever asked. Rules out it running the app — it asks for a screenshot or for
  `webtester` instead.
- **2026-08-29 — Push follows every commit, automatically.** Replaces "push only if
  asked", written the same day: an approved commit is already the user's decision, and a
  commit that exists on one machine only is lost with an ephemeral container. The
  approval gate stays on the commit, not on the push. Rules out pushing a branch other
  than the current one, and rules out `--force` entirely.
- **2026-08-29 — Commit messages are written in Catalan, always.** Replaces the rule
  written earlier the same day, which followed whatever language the repository's history
  already used. That treated the language as a property of the repo; it is a property of
  the person keeping the history. Rules out switching language per project, including
  this one, whose code and documents are in English.
- **2026-08-29 — No attribution trailer in commit messages, without exception.**
  Replaces the earlier wording, agreed the same day, that let a harness policy requiring
  a trailer win: that exception was the environment overruling the kit through the back
  door, which the decision below rules out. The repository's owner decides what its
  history looks like.
- **2026-08-29 — The plugin version gets bumped with every shipped change.** `claude
  plugin update` and auto-update compare `version` in `plugin.json`, not commits, so at
  0.1.0 every existing install silently kept the copy it first fetched. Cloud sessions
  hid it: their setup script installs fresh rather than updating.
- **2026-08-29 — The user's rules outrank the environment's.** This session's harness
  instructs the agent to commit and push its work; the kit says never to commit unasked.
  The kit wins: work waits in the working tree with its message proposed. Rules out
  treating a `Stop` hook's reminder, or any automated nudge, as permission. The cost is
  accepted: unpushed work dies with an ephemeral container, so commits get proposed at
  natural checkpoints rather than accumulating.
- **2026-08-29 — The SVG icon flow is not coming into the kit.** What travels (raster to
  stroke, flattening, normalisation) is thin next to what stays with the project — the
  house style and the approved set — and it would have brought native dependencies and a
  per-machine install for it.
- **2026-08-29 — The collaboration method is split, not a skill of its own.** Inclination
  versus decision goes inside `project-plan`; the critical-librarian posture goes in the
  `project-template` fragment because it must always apply; a `docs/` skeleton waits until
  one file is genuinely not enough. Rules out a second skill owning the same documents.
- **2026-08-29 — "Never commit unasked" is a CLAUDE.md fragment, not a hook.** A
  `PreToolUse` hook cannot know whether the user asked, so it could only escalate to a
  permission prompt; the skill already requires proposing messages first. Revisit with
  evidence if it turns out to be ignored in practice.
- **2026-08-27 — The plan keeps no "done" section.** Finished work becomes a
  decision, a constraint in `CLAUDE.md`, or nothing at all; the diff is the record.
  A done list is the section that rots first and drags the rest of the file's
  credibility with it. Decisions are superseded, never edited, borrowed from ADRs.
- **2026-08-27 — Principle 6 judges scope, not line count.** A line budget rewards
  dense prose over clear prose and invites splitting to satisfy a number.
