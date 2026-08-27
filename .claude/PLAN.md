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
- **2026-08-27 — Principle 6 judges scope, not line count.** A line budget rewards
  dense prose over clear prose and invites splitting to satisfy a number.

## Done

- **Canary skill removed.** The install path is proven end to end; keeping it cost
  ~110 tokens per session for a question nobody asks.
