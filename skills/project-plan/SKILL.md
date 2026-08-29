---
name: project-plan
description: Keep one living document per project holding what is pending and what is still undecided, and keep it true. Use when picking up a project and needing to know where it stands, when a piece of work is finished, when a decision is taken or deliberately deferred, when new work is discovered mid-task, or when someone asks what is left. Covers finding the document a project already has, choosing how to order it, writing entries that stay one line, justifying a decision at the moment it is taken, and pruning so the file does not rot into a log nobody trusts. Not a replacement for an issue tracker the team already uses.
---

# The project plan

One document per project, holding the two things nothing else keeps: **what is
pending** and **what is still undecided**. The code says what was built and git says
what changed; neither says what was chosen over what, or what is waiting on whom.

It is only worth having if it is true. A stale plan is worse than no plan, because
someone acts on it.

The design constraint is that **someone arriving cold — a person after two months, or
an agent with no history — can read the whole thing in under a minute and know what
to do next.** Every rule below follows from that, including the ones that feel severe.

## 1. Find it before creating one

Look for what the project already uses — `docs/todo.md`, `ROADMAP.md`, `PLAN.md`, a
list in `README.md`. Adopt it and improve it in place. A second planning file is how
a project ends up with two, both half true.

Only if there is none, create `.claude/PLAN.md`.

## 2. Choose how to order it, then leave it alone

There is no fixed structure. Pick the one that answers *what do I do next* here:

| The project is | Order by |
|---|---|
| Phased, with a release or dates | Time — now, next, later |
| Blocked by dependencies between its parts | Layer — data, API, interface |
| A flat set of independent work | Impact — what hurts most, first |

Choose once. Re-sorting the document every few weeks destroys any sense that it
reflects something real.

## 3. What goes in it

**Pending work.** One line each, imperative. A second line only when the reason is
not obvious from the first.

**Open questions.** What must be decided before something else can move. Each says
what it blocks and what would settle it. A question that blocks nothing is not an
open question, it is an opinion — leave it out.

**Decisions.** Dated. One line for what was decided, one for why, and where it
matters, one for what it rules out. This is the part that earns the file: the reason
is what stops the same argument being had again in three months.

**An inclination is not a decision.** While a question is still being explored, it stays
in Open questions — "leaning towards A", "A or B, unresolved" — however clear the
direction feels. Only what was explicitly closed goes in Decided. Writing a preference
down as settled is how a conversation gets a conclusion nobody actually reached.

An approach that was **tried and failed** is a decision too — the most valuable kind,
because without it someone tries it again. Record what was attempted and what it cost.

**Decisions are never edited.** When one is reversed, add a new dated entry that names
the one it replaces, and mark the old one superseded rather than deleting it. Editing
in place destroys exactly what the file was for: why the other thing was chosen at the
time. (This is the one rule worth borrowing wholesale from Architecture Decision
Records. When a project accumulates enough of them, graduate to `docs/adr/` — one file
per decision — and leave the plan holding only pending work and open questions.)

**Finished work does not get a section.** It either produced knowledge — in which case
it is a decision, or a constraint that belongs in the project's `CLAUDE.md` — or it did
not, in which case the diff is the record and the entry is deleted. A "done" list is
how these documents die: it grows, nobody reads it, and its dead weight makes people
stop trusting the sections that matter.

## 4. When to write

In the same turn as the work. Not at the end of the session, not when asked. An
entry written later is written from memory, and it is the *why* that gets lost first.

- **Finished something** → delete the entry, unless it taught something; then write
  that as a decision or a constraint first.
- **Took a decision** → record it with its reason before moving on. A decision
  without a reason is not a decision; it is a fact that will be re-litigated.
- **Deferred a decision** → it becomes an open question, with what it blocks. A
  deferral that is not written down is indistinguishable from an oversight.
- **Leaning one way** → that is an open question with a note on the leaning, not an
  entry in Decided.
- **Found new work mid-task** → write it down instead of doing it. That is most of
  the value: the plan is where scope goes to wait instead of growing the change in
  front of you.

## 5. Keep it short

If reading the whole file takes more than a minute, it has stopped being a plan and
become a backlog — and a backlog belongs in an issue tracker, not here.

- One line per entry. No code, no error output, no pasted context: point at the file
  or the issue instead.
- Prune **Done** when it stops being read. Decisions stay: they are the expensive
  part. Pending work and open questions are *state* — when they are no longer true,
  they are deleted, not archived.
- Never leave an entry that is neither current nor deleted. "Maybe later" items are
  the ones that make people stop trusting the file.

## What goes somewhere else

The file survives only if this boundary is enforced. Everything below is a real thing
worth writing down, and none of it belongs here:

| That | Goes to |
|---|---|
| How to work in this repo, conventions, invariants, a constraint discovered the hard way | the project's `CLAUDE.md` |
| A bug, or work with an owner and a due date | the issue tracker |
| What shipped, and when | the changelog, or git |
| The full argument behind a large decision | a design document, linked in one line from here |

## Not this

No owners, no status fields, no priorities, no estimates, no risk register, no
tech-debt section. Each is an issue tracker growing inside a markdown file, and a
document with fields is a document nobody updates. If the project genuinely needs
those, it needs a tracker, and the plan shrinks to what a tracker is bad at: what is
undecided, and why things were chosen.
