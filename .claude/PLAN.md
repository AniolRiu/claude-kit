# Plan

Ordered by impact: this is a flat set of independent work, not a phased project.

## Pending

- Decide whether `uix-expert` writes its findings into the project it reviews.
  Blocked on the open question below; nothing else waits on it.
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
