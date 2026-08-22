---
name: ui-ux
description: Review or design an interface from the point of view of the person using it — hierarchy, states, copy, spacing and accessibility. Use when asked to review a screen or a flow, to design one, to judge whether a change makes a product easier to use, or to give a design opinion on code that already works. Also use before shipping a user-facing feature that has only ever been judged on whether it functions. Produces a few findings of real impact, each naming the problem it solves, not an exhaustive list of details. Not for implementing the changes, and not for visual polish requested as an afterthought.
---

# UI/UX: judging an interface as the person using it

This is a different mode of looking from programming. While writing the code, the
question is "does it work"; here that question is already answered and it is the
wrong one. The question is what the interface asks of the person in front of it,
and whether it answers back.

Hold that mode. The moment the review turns into "the component could be split in
two", it has stopped being a review.

## 1. Look before reading

Establish what the screen is for before opening the files that build it:

- What is the person trying to do here, and what did they arrive from?
- What is the one thing this screen wants them to do?
- What can go wrong, and how would they find out?

Then read the UI code to answer those, not to inventory it. Read the states, the
copy and the labels first; the styling last. If the app can be run, seeing the
screen beats inferring it — a layout that reads fine in JSX can still be a wall.

## 2. What to examine

**Hierarchy.** One primary action per screen, and it should be the most prominent
thing. If three buttons carry equal weight, the design has not decided. Rank what
is on the screen by importance and check the visual weight matches.

**States.** Every screen has more than the happy one: loading, empty, error,
success, no-permission, partial data. The empty and error states are where
products usually fall apart, and they are the states nobody looks at because
seeded data hides them. Ask what each looks like; if the answer is "nothing" or "a
spinner forever", that is the finding.

**Copy.** Labels use the words the person uses, not the schema's. A button says
what will happen when it is pressed ("Delete 3 invoices", not "Confirm"). An error
says what to do next, not what failed internally. Anything that says "an error
occurred" is a placeholder that shipped.

**Accessibility.** Controls have real accessible names, not just placeholders.
Order of focus follows order of meaning. Colour is never the only carrier of
information. Targets are big enough to hit, contrast is enough to read. These are
not a separate checklist — an element that cannot be reached by role or label is
usually badly labelled for everyone.

**Rhythm and grouping.** Space communicates relationship: things that belong
together sit together, and unrelated things are not equidistant. Consistency
across the app beats local perfection on one screen.

**Forms, when there are any.** When validation fires, where the error appears
relative to the field, whether the work is lost on failure, and whether a
destructive action can be reached by accident or undone after it.

## 3. What to report

**Few things, ranked.** Three to five findings. A list of thirty nits does not get
acted on; it gets skimmed and closed. If something small is genuinely worth it —
one word in a button — it can be a finding, but it competes with the others.

Each finding has three parts:

1. **What is there.** Concretely, so it can be found: the screen, the element.
2. **The problem for the person using it.** Who gets stuck, and on what. This is
   the part that justifies the change; a finding without it is a preference.
3. **A concrete alternative.** The actual replacement — this copy, this order,
   this state — not the principle it follows. "Improve the hierarchy" is not a
   finding. "Make Save the only filled button and demote Cancel to a link" is.

Then:

- **State the assumptions** made about who the user is, what they know, and what
  device and context they are in. Most disagreements about interfaces are
  disagreements about the user, and they are cheaper to settle when stated.
- **Separate what changes behaviour from what changes appearance.** They have very
  different costs, and mixing them makes the whole list feel expensive.
- Say plainly when something is already right and load-bearing, so it does not get
  refactored away. One line, not a paragraph of praise.
- Say when there is not enough to judge on — an unseen state, an unknown user, a
  flow whose start is elsewhere — instead of guessing.

## 4. When designing rather than reviewing

Same discipline, run forwards:

- Start from what the person is trying to do and what they bring with them, and
  say those assumptions out loud before proposing anything.
- Describe the screen as hierarchy and states — what dominates, what each state
  shows — before any visual detail.
- Propose one design and say what it is trading away. Three options with no
  recommendation is a way of not deciding.
- Design the error and empty states in the same pass as the happy path. Retrofitted
  ones are how products end up with a spinner and no way out.

## 5. Out of scope

Do not implement the changes; the point is the second pair of eyes, and the code
belongs to whoever asked. Do not redesign the feature into a different feature —
the subject is the screen as it exists and the job it has. Do not invoke a
principle without the concrete change it implies. Do not pad the list to look
thorough.
