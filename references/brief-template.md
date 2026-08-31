# Writing the brief, and running a queue

## Why the brief carries everything

The worker sees only the text you send plus what it can read from the working
tree. No chat history, no shared context, and in a headless run no way to ask -
there is nobody there to answer. A question the worker cannot ask becomes a
guess, and a guess becomes work you have to throw away.

So the test for a brief is not "is this clear to me" but "could a competent
stranger execute this having read nothing else."

## Structure

Use headed sections. What matters is that every part a worker must act on is
visually separable and individually addressable, so a correction can name one
section instead of resending the whole thing. Plain capitalized headers do this
as well as any markup, and unlike a specific syntax they don't invite the worker
to treat the format as the point. Some models handle one delimiter style better
than another; the discipline is what transfers, not the brackets.

## Template

```
TASK: <one sentence - what to produce, not how>

You are doing this work yourself. Do not describe what should be done; run the
commands and write the files.

CONTEXT
<What the worker needs to know that it cannot read off the tree. Absolute
working directory. Any file, doc, or skill to read first, BY PATH - it will not
guess. State the current condition of anything it will change.>

WHAT TO DO
<Numbered steps where order matters. Exact commands where the command matters -
including anything that has to be verbatim.>

SUCCESS CRITERIA
<What done looks like, concretely enough that the worker can check itself.>

BOUNDARIES
<What it may create, edit, or run. What it must not touch, named explicitly.
Whether it may commit, install, or reach the network - by default, no.>

IF YOU GET STUCK
<Say to stop and report rather than improvise around a blocker. A worker that
invents a way past an obstacle produces work nobody asked for and a report that
hides it.>

FINAL REPORT
<The exact sections you want back. Ask for evidence - commands run and their
results, files touched, anything it could not do.>
```

## Things that go wrong

- **Skills and docs referenced by name, not path.** The worker cannot search
  your machine's conventions. Give the absolute path.
- **Success criteria that only you can evaluate.** "Make it good" gives the
  worker nothing to check itself against, so it stops when it feels done.
- **Boundaries stated as intent rather than paths.** "Don't break anything" is
  not a boundary. "Do not modify `<path>`" is.
- **Silence about commits and network.** State the default explicitly. A worker
  that commits when you meant it not to has mixed unreviewed work into history.
- **A report format that asks for a narrative.** Ask for a list of commands and
  their results; a narrative is where an unperformed step hides.
- **The brief passed as a command argument.** Write it to a file and send it on
  stdin - `opencode run ... < brief.txt`, or `Get-Content brief.txt -Raw |
  opencode run ...` in PowerShell. An argument is parsed by the shell before
  opencode sees it, so quotes, newlines and `$` are at the mercy of quoting
  rules. `"$(cat brief.txt)"` does not avoid this: it is command substitution,
  and the text still goes through argv. A mangled instruction is invisible until
  the output is wrong.

## Running a queue

Sequential by default, one reviewed unit at a time. Parallel workers on one tree
collide, and a failure halfway through a batch leaves you unable to tell which
results are trustworthy.

- **Review each task before dispatching the next.** The point of a queue is that
  errors don't compound; that only holds if you actually look in between.
- **Carry decided constraints forward.** A decision made in task 2 - a naming
  convention, an interface shape, a rejected approach - belongs in the brief for
  task 3. The worker's session may remember it; a fresh session will not.
- **Keep one task per brief.** Two tasks in one brief produce a report where you
  cannot tell which half failed.
- **Do a coherence check at the end.** Individually correct steps can still add
  up to something inconsistent, and only a look at the whole result finds it.
- **Stop the queue on a scope change.** If correct completion needs going beyond
  the brief, ask - don't expand the mandate yourself and don't carry the
  expansion into the remaining tasks.
