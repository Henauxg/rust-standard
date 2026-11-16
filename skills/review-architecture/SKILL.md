---
name: review-architecture
description: Evaluate a crate's architecture in depth, read cold, after many iterations have grown it. Does the structure still fit what the crate does, what consumes it and where it is going? Looks at boundaries and dependencies, where responsibilities sit, duplication, the data model, completeness, errors, state, performance by design, testability, readiness for the next step, and size. Confirms each finding on the running code, ranks them, proposes a plan, and discusses it with the user before anything changes. Use when asked for an architecture review, a deeper crate evaluation, or "did we get the architecture wrong". Bugs are the review skill's job, style the review-standard skill's.
license: MIT, in LICENSE.txt
---

# Review a crate's architecture

This skill steps back from the latest change and asks whether the crate's
structure still fits its job. It is meant for after many iterations, when
each fix was reasonable on its own and the sum may not be. It changes no
code: it ends with a ranked report and a plan the user agrees to.

Bugs are out of scope (`/review`), and so is the coding standard
(`/review-standard`). A bug or a rule broken goes in only when it shows a
structural cause, such as the same check written twice and already
disagreeing.

## Arguments

`/review-architecture [path] [focus]`

- `path`: the crate to review, such as `crates/parser`. Without it, ask.
- `focus`: any other text names what worries the user most. Read that
  first, and still cover the rest.

## 1. Read what the crate is for

Before the code, read what says what it must do and where it is going:

- the specs it implements, often in `docs/`;
- its design and progress notes, and the notes left for the next session;
- the project's `AGENTS.md` or `CLAUDE.md`;
- the coding standard's `organization.md`, for what a module should hold
  (`/review-standard` says where the standard is).

Write down, in a few lines: what the crate takes in, what it produces, who
consumes the output (another crate, a binary, a user, the network),
and the next large planned step. A structure is judged against these.

## 2. Read the code cold

- Read the committed tree, not your memory of building it. If another
  session may be editing the working tree, extract `HEAD` to the
  scratchpad first:

  ```sh
  git archive HEAD <path> | tar -x -C <scratchpad>/review
  ```

- The reading is long. Run steps 2 to 5 in one subagent with a fresh
  context, give it these steps, and keep only its report.
- Map the crate before judging it:
  - its stages or layers and the data each passes to the next;
  - its public API and who calls it;
  - which module imports which (`grep -rn "use crate::"`), and any import
    that goes against the order of the stages;
  - the size of each file (`wc -l`), and the largest functions.

## 3. Look through each lens

Each lens is a question to ask of the whole crate. For each, look for the
concrete instance, with file and line. The examples in parentheses come
from past reviews. Skip a lens that does not apply,
such as concurrency in a crate with no threads, and say so.

1. **Boundaries and dependencies.** Do the modules form layers, and does
   every import go the way the layers do? Look for cycles, a later stage
   imported by an earlier one, internals reached through the public API,
   and external crates heavier than what they are used for.
2. **Responsibilities.** Can a stated rule place each kind of logic, and
   does the code follow it? Look for logic in the wrong layer, modules
   that mix concerns, and helpers that belong to a shared lower layer.
   (A validation living in the parser; no written rule at all.)
3. **Duplication.** Is any lookup, rule, index or conversion written more
   than once? Compare the copies: they usually already differ. (A
   reference resolved in three places, field indexes built eight times.)
4. **The data model.** Do the types fit the domain and the consumers? Look
   for invalid states the types allow, enums with variants for unrelated
   purposes, flags that switch a mode, made-up values standing in for a
   missing one, identity by position where consumers need stable ids, raw
   input kept where a resolved value is owed, and types from an earlier
   stage leaking into the output.
5. **Completeness.** Does every kind of input reach the code that handles
   it? List the kinds the crate accepts and trace each one. Coverage that
   depends on two lists staying in step leaks, and a `_` arm can swallow
   a kind added later. (A kind of input that passed with no check.)
6. **One input, several contexts.** Where something is computed once per
   context (per caller, per subject, per kind), is each result kept, and
   is each finding reported once?
7. **Errors and reporting.** Does every failure go through one path, with
   one wording, to someone who can act on it? Look for errors swallowed,
   reported twice, or worded differently for the same cause.
8. **State and ownership.** Where does mutable state live, who changes
   it, and in what order? Look for shared state threaded through many
   layers, caches that can go stale, and clones or reference counting
   standing in for a clear owner.
9. **Performance by design.** Is the work bounded by the structure, or
   by fixes added one input shape at a time? Look for indexes rebuilt per
   use, work repeated per context, and quadratic loops over sizes the
   crate sees. Prefer indexes built once with the data, and counters a
   test can compare at n and 2n.
10. **Testability.** Do the tests pin the rules the crate must keep, and
    can a part be tested without the whole? Look for rules with no test,
    tests that only check the code runs, and seams missing where a test
    needs one.
11. **The next step.** Walk through the next planned step, or the next
    likely feature, against the current structure. What would it have to
    change, work around, or duplicate? An assumption built in everywhere
    is a finding when that step breaks it.
12. **Size and names.** Which files and functions are far above their
    neighbours, and along which cut would they split? Do the module and
    type names still match the concepts the crate works with?

Also list what is sound: the parts a fix must not disturb.

## 4. Confirm each finding

A finding is a concrete consequence, not a smell. For each one:

1. Try to prove it wrong: a check elsewhere, a caller that never passes
   that input, a type that rules it out. If one exists, drop the finding.
2. Show it on the running code where that is cheap: the smallest input
   that passes when it should fail, reports twice, or takes quadratic
   time, run through a test or the crate's command-line tool. Keep these
   inputs in the scratchpad and name them in the report.
3. Mark it **confirmed** (shown on the code) or **read** (from reading
   only).

## 5. Report

Findings in order of value, each with:

- what is wrong, in one or two sentences, with file and line;
- the consequence, concrete: the input and the wrong result, the repeated
  work, the copy that disagrees;
- the evidence: confirmed with which input, or read;
- the fix, and its size: small, medium or large;
- when to fix it: now, before a planned step (name it), when the area is
  next touched, or leave it.

Then "Fine as is", then a proposed plan: the fixes in the order they
should land, each fix that unblocks another first.

Write it plainly, for the user to decide from.

## 6. Discuss, then record the plan

- Change no code during the review.
- Give the user the report, then ask about each fix that needs a
  decision, one at a time, with your recommendation first. Explain any
  term the user questions, such as what a work counter is.
- Record what the user approves where the next session will find it, such
  as the project's to-do or handoff notes: the approved fixes in their
  order, and the findings left for later. Commit that note.
- Each approved fix is then its own piece of work, verified and committed
  on its own. A split of a large file is a move-only commit, checked by
  comparing the old file's tokens with the new files'.
