---
name: commit
description: Consolidate work into commits — read the diff before writing anything, clean what should not ship, check it builds, split by reason for change, and propose every message for approval before running git. Use when asked to commit, when deciding how to split a set of changes, when writing or rewriting a commit message, or when the working tree has grown enough to be worth consolidating. Not for deciding *whether* to commit: that is the user's call and is never inferred from a request to write, fix or document something.
---

# Committing

Consolidating is not part of doing the work. While iterating, changes stay in the
working tree — they can be built and deployed from there, because deploying commits
nothing. When enough has accumulated to be worth consolidating, say so and offer;
do not decide it alone.

**Never run `git commit` on your own initiative.** And a permission is spent on the
commits proposed at that moment: it does not stay open for the next ones. If the word
*commit*, or an unmistakable equivalent, has not been said, nothing is committed —
the work stays in the working tree and you say what is pending there.

## 1. Look at what actually changed

`git status`, `git diff`. Write the message from the diff, not from memory of what you
set out to do: the two differ more often than feels possible, and the diff is the one
that will be read later.

## 2. Clean before you stage

Debug logging, commented-out code, temporary files, throwaway scripts, anything from a
scratch directory, generated files. `git status` should hold no surprises for the person
approving it.

## 3. Check it builds

Using the project's own build command. A commit that does not build breaks any future
`git bisect` — the tool is only as useful as the worst commit in the range. Where a
production build is separate from the development one, the production build is the one
that counts: some errors appear only there.

## 4. One commit, one reason to change

A refactor and a behaviour change go in separate commits, so each can be reviewed and
reverted on its own. Do not split for the sake of splitting: two commits that must land
together are one commit.

## 5. The message

```
type(scope): short title in lower case

Body: what changed and, above all, WHY.
```

- Title around 70 characters, no full stop. `feat`, `fix`, `refactor`, `docs`, `chore`.
- **The why is the whole point.** The diff already says which files changed; the reason
  is unrecoverable if it is not written down. If the change corrects an earlier wrong
  assumption, say so outright — that is the most valuable thing a message can contain.
- Proportionate. A one-line fix does not need twenty lines of description. Facts, not
  adjectives.
- The test: the change should be understandable **without opening the diff**.
- Write in the language the repository's history already uses. English when the history
  is empty or mixed.
- **No `Co-Authored-By:` trailer.** It says nothing the message does not and clutters the
  history: the message explains the change, not who typed it. The exception is a repo
  convention or a harness policy that requires a trailer — those win, and are the only
  thing that does.

## 6. Propose every message before running anything

Always, however small the change, and wait for approval. When the work splits into
several commits, propose **all of them at once**, each with the files it takes: that is
the only way to see the whole split and regroup it before it is written into history.

## 7. After

The working tree is clean and you have verified it. `push` only if asked.

Never `--no-verify`, never skip a hook: if a hook fails, the cause gets fixed. Prefer a
new commit over amending an existing one — amending rewrites history that someone may
already have.
