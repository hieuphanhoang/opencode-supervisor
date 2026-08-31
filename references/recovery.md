# When a run misbehaves

The rule underneath all of these: **a run is finished when the process has
exited, not when something says it is.** Read the tree and the capture, not a
status line.

## The run comes back empty

No final text, or a final message that says nothing. Check in this order:

- Did the process exit non-zero? A usage error - bad flags, an empty brief -
  fails before the model is ever called.
- Does the capture contain any `tool_use` events? If none, the worker never
  started work, which usually means the brief didn't reach it intact. Suspect
  shell quoting first; re-send from a file.
- Did it run out of context? A brief plus large files can exceed the model's
  declared window, and the truncation is silent. Check the config's declared
  limit against what the task actually had to read.

An empty result is not a result. Don't summarize it as "the worker found
nothing" - say the run produced nothing and why.

## Unknown model or agent

`provider/model` must match `opencode models` exactly, including the tag
(`:latest` is part of the name, not an optional suffix). A model installed in
ollama but not declared in the opencode config will not resolve however
correctly you spell it.

For agents, `opencode agent list` is the authority. An agent name that exists on
one machine may not exist on another - agent profiles come from local config.

## The run hangs

Almost always a permission prompt nobody can answer. A headless run without
`--auto` blocks on any action the agent's config marks `ask`.

- Check the capture for a pending permission request naming a path or command.
- Fix it by scoping permissions in the opencode config for that path, or by
  narrowing the brief so the worker doesn't need the action.
- `--auto` is the blunt fix and carries the trust escalation described in the
  main skill. Don't reach for it just to unstick a run.

If there is no pending prompt, a local model may simply be slow - check whether
tokens are still being produced before concluding it's stuck.

## Timeout, or you killed it

**Assume partial edits.** A worker interrupted mid-task has written some files
and not others, and nothing marks which.

Before retrying or cleaning:

1. Look at the working tree and compare against the pre-dispatch state you
   recorded. This is why the dirty-tree check happens before dispatch.
2. Decide deliberately whether to keep or discard the partial work. Retrying on
   top of half-finished edits produces a result neither you nor the worker can
   explain.
3. If you discard, do it by reverting the specific paths - not by cleaning the
   whole tree, which takes pre-existing changes with it.

Then retry with a brief that accounts for the current state, rather than the
original one that assumed a clean start.

## The worker wrote outside its boundaries

Have it recover inside the authorized workspace rather than deleting external
paths yourself. You may not know what was already there, and a path outside the
working directory may hold something that was never the worker's to touch.

Tell the user what happened and where, even if the recovery is clean. A boundary
breach is information about the model, and it's exactly the kind of thing a
summary quietly drops.

## The worker's report and the tree disagree

Believe the tree. Then treat the discrepancy as a finding in its own right and
report it - a worker that misreports its own work will do so again, and that
matters more to the user than this one task.
