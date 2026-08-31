# When the delegated task touches code

Everything here applies only when the worker changed a working tree. For tasks
that produced a document, a decision, a benchmark, or a diagnosis, the
equivalent is verifying the artifacts against the stated success criteria -
don't manufacture gates a task never had.

## Read the diff

Start from the files the worker touched, then read the change against the brief:

- Did it do what was asked - nothing more, nothing less?
- **Scope creep** is the common failure: a tidy-up, a rename, a "while I was in
  here" refactor. Each is defensible alone; together they make a diff nobody
  wants to review and a change nobody can revert cleanly.
- Are there edits outside the paths the brief authorized?
- For removals, grep for dangling references. For schema or migration changes,
  round-trip them.

A diff that is larger than the task suggests the worker did not understand the
task. That is worth knowing before you look at whether the code is correct.

## Re-run the project's own checks

Discover the project's real check commands - `AGENTS.md`, `CLAUDE.md`, the
Makefile, the package scripts, CI config. Do not assume `npm test` or `pytest`;
running the wrong command and getting a pass is worse than running nothing,
because now you believe something.

**Never accept "gates passed" on faith.** Run them yourself. The worker's report
is a claim about the world, and the whole reason you are the reviewer is that
claims need checking.

## Review test edits before trusting green tests

If the worker changed tests as well as code, the gates prove less than they
appear to. Green tests after the worker moved the goalposts tell you the worker
is self-consistent, not that the code is right.

Look specifically for: assertions loosened or deleted, cases skipped or marked
expected-failure, fixtures edited so a broken path is no longer exercised, and
new tests that only assert what the new code happens to do.

A test change is legitimate when the brief asked for a behavior change and the
test encoded the old behavior. It is not legitimate as a way to make a failure
go away. If you cannot tell which happened, that's a retry.

## The commit boundary

**The worker edits; the supervisor commits.** Committing should be the act of the
party that verified the work - a commit by the author of the change records
nobody's judgment but their own.

Only commit after the checks pass and the diff holds. Then:

- Commit the verified work yourself, with a clear message describing what
  changed.
- Never `git commit -a`. It sweeps up whatever else is in the tree, including
  work you did not review and, on a shared repo, work another member is
  half-way through writing.
- Commit the worker's paths deliberately, not the tree indiscriminately.

**Pushing is the owner's call**, and so is opening a PR, creating a repository,
or making one public. A commit is local and reversible; publishing is neither.
Get the go-ahead at the moment of acting, not once in advance.

If the change needs rework, send a delta brief into the same session rather than
restating the whole task, and review again.
