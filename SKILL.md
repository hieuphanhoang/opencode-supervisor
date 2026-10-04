---
name: opencode-supervisor
description: Delegate a task to a model running under opencode and supervise it end to end - preflight the CLI, choose a suitable worker model (or let the user choose), turn the request into a brief the worker can execute blind, dispatch it headlessly with structured capture, verify the result against what was actually asked, retry narrowly when it falls short, and report with authorship kept straight. Use this whenever the user wants another model to do the work instead of you, such as "have opencode do X", "delegate this to a local model", "get qwen/ollama to build this", "run this through opencode", "supervise the model while it does Y", "pick the best model for this task", or any request to delegate, orchestrate, oversee, grade, or retry another agent's output. Also use it when a task is explicitly framed as "the model does it, not you".
license: MIT
metadata:
  short-description: Delegate, supervise and verify opencode worker tasks
---

# Opencode Supervisor

Three roles are in play, and keeping them straight is what makes this work:

- **Supervisor** - you, whatever host you're running in (Claude Code, Codex, Cursor, ...). You shape the work, dispatch it, judge the result.
- **Harness** - the `opencode` CLI. It runs the worker, holds sessions, enforces permissions.
- **Worker** - the model opencode is driving (`provider/model`). It does the actual task.

The point is that the worker does the work and you do not. If you quietly
complete the task yourself because it seemed faster, the user got nothing they
asked for - they wanted to see a particular model attempt a particular job.
Deliver the worker's output, clearly labelled as the worker's, plus your
judgment of it.

Not every delegated task is a coding task - some produce documents, run a
protocol, or benchmark a model. Where a step below applies only to work that
touches a git tree, it says so; demanding a diff and green tests from a task
that has neither is how a supervisor asks for evidence that cannot exist.

## Preflight

Check once per session, before promising the user anything:

These three are shell-neutral:

```
opencode --version      # installed, and which version
opencode agent list     # configured agents and their permissions
opencode models         # what this install can actually reach
```

Locating the binary is shell-specific, and so is probing a local server. Use the
form that matches the shell you are actually in - `command -v` is not a
PowerShell command, and in Windows PowerShell `curl` is an alias for
`Invoke-WebRequest`, not curl.exe, so the flags differ:

| Check | POSIX (bash/zsh, Git Bash) | Windows PowerShell |
|---|---|---|
| Which binary | `command -v opencode` | `(Get-Command opencode).Source` |
| Local server up | `curl -s http://localhost:11434/api/tags` | `Invoke-RestMethod http://localhost:11434/api/tags` (or `curl.exe -s ...`) |

The worker needs a reachable provider, which is either cloud credentials
(`opencode auth list` shows at least one) **or** a running local provider. Do
not demand auth: a purely local ollama setup has no credential at all, and a
preflight that insists on one fails a perfectly good install. `ollama list`
shows what is installed locally, but see step 2 - installed is not the same as
reachable by opencode.

## Workflow

1. Understand the goal and what "done" looks like.
2. Resolve the model policy, then confirm the worker model with the user - or
   choose it yourself, if they have granted that.
3. Write a brief the worker can execute with no other context.
4. Check the tree is clean, then dispatch with structured capture.
5. Watch for the final result; ignore the noise in between.
6. Verify the result yourself. Do not accept the worker's self-report.
7. Retry narrowly if it fell short, or report.

## 1. Understand the goal

Ask the user only what you cannot infer, and ask it now rather than mid-run - a
local model can take many minutes per attempt, so a question you skip becomes a
wasted run. Worth pinning down: the working directory, what the worker may
create or modify, what counts as evidence it worked, and what is off-limits.

## 2. Choose the model

Model choice drives cost, speed, and whether the task is feasible at all.

Separate two questions that look alike. **What can this install run?** is a
capability question, answered by `opencode models` and the opencode config
(`~/.config/opencode/opencode.jsonc`, plus any project-level `opencode.json`) -
which declares context limits, tool-call support, and often the author's notes
on which quantization fits in VRAM. **Which of those may I spend on this task?**
is a policy question, and a config that merely declares a model is not an answer
to it. Declaring a model makes it available; it does not make it approved, and
treating the catalog as an allowlist is how a supervisor reaches for a metered model
nobody authorized.

So consult capabilities always, and resolve policy in this order, stopping at
the first that actually states one:

1. **A standing delegation policy in the opencode config**, if one is written
   there. That is a good home for a machine-local policy, since it sits beside
   the capability data it refers to. Only an explicit policy or allowlist counts.
2. **A repository policy** - `AGENTS.md` or `.agents/`. Legitimate in its own
   right: which models a project permits is a different question from what a
   machine can run, and a repo may restrict further than the machine does.
3. **A per-run instruction from the user**, including an instruction to choose
   for them - see delegated choice below.
4. **Ask the user.** The default when nothing above grants otherwise. Guessing
   from the catalog risks a metered model and a surprise bill, and the user
   usually knows things you don't - which model they're benchmarking, what fits
   in VRAM, what they already pay for.

Whichever route, cross-check local models against the harness, not the machine:
a model installed in ollama is invisible to opencode unless the config declares
it, so offering one that won't resolve wastes a run. Tell the user about any gap.

### Hard gates

These eliminate candidates. A model that fails one is not a candidate at any
score, so apply them before comparing anything:

- **Reachable by opencode.** It must appear in `opencode models`. Installed
  locally is not the same as reachable.
- **Tool calling**, for any task that edits files or runs commands. A model with
  `tool_call: false` cannot do agentic work at all.
- **Declared context exceeds the estimated need, with headroom.** Overflow is
  silent, so a near-miss is a failed run rather than a degraded one.

### Ranking what survives

Fit first, then rank - because "best" depends on what the task is:

- **A code or filesystem task** wants reasoning depth and enough context.
- **A critique, summary, or diagnosis** may rank differently; raw parameter count
  matters less than instruction-following.
- **A benchmark or evaluation names its own model.** Then that model is the
  subject of the experiment, not a candidate to be optimized away.

Among the models that fit: parameter count beats quantization for multi-step
reasoning, but a model that spills out of VRAM to CPU can be several times
slower - say so when it is a real tradeoff. Very small models (about 7B and
under) are smoke tests, not workers; they declare tool support and still lose the
thread on step four.

### Delegated choice - when the user says "you pick"

The user can hand you the choice, for one run or as a standing policy. That is a
grant, and it has to be given: you may never infer it from a capability catalog,
because "this machine can run it" was never an answer to "you may spend this."

When it is granted:

- **Choose inside the approved set only.** Absent an explicit policy saying
  otherwise, that set is the free and local models. Delegated authority over
  models that cost nothing is a much smaller grant than authority over an account
  with billing attached, and a user who says "you pick" is rarely also saying
  "spend what you like."
- **Never override a model the user named** when the point of the task was that
  model. Quietly upgrading a benchmark to a stronger model destroys the
  experiment while looking like good judgment, and you will produce a confident
  justification for having done it.
- **Break ties deterministically** inside a cost class - usually the faster or
  cheaper candidate, since being wrong with it costs less. Report the runner-up.
- **Stop and ask only when the tie crosses a boundary with consequences**: paid
  against free, local against cloud where the data is sensitive, slow-and-capable
  against fast-and-weak, or nothing passing the hard gates at all. Asking on
  every tie would defeat the point of the mode.
- **Report the choice with its reasoning and the runner-up.** An auto-selection
  nobody can audit is indistinguishable from a guess, and the reasoning is what
  lets the user correct the policy rather than just the outcome.

## 3. Write the brief

The worker doesn't have the conversation you just had, and cannot ask - a
headless run has nobody to answer it. Everything the task needs goes in the
brief. Rewrite, don't forward.

Asking for a specific final-report format pays for itself twice: the worker
converges instead of trailing off, and you get something reviewable rather than
a wall of tool output.

Keep scope honest. If the user asked for the worker to be *prompted*, don't
expand that into doing the task yourself. If they asked for a *review*, judge
the deliverable, not every command in the transcript.

Structure and a reusable template: `references/brief-template.md`.

## 4. Dispatch

**Check the tree first.** If the working directory is already dirty, record what
was pre-existing. Otherwise you cannot tell the worker's edits from changes that
were there before, and the review in step 6 has no baseline.

Capture the run as structured events rather than formatted text. The default
output carries ANSI escape codes and is built for a human watching a terminal;
`--format json` emits JSONL you can actually inspect - session id, each tool
call with its command and exit code, per-step token counts and cost, and the
final text.

**Send the brief on stdin, not as an argument.** `opencode run` reads the prompt
from stdin when no message argument is given, and that is the only delivery path
that cannot be mangled: an argument is parsed by the shell first, so quotes,
newlines, `$`, backticks and semicolons in a brief are at the mercy of quoting
rules that differ between bash, PowerShell and cmd. `"$(cat brief.txt)"` is
command substitution, not piping - the text still passes through argv and is
still exposed to that parsing. Stdin bypasses it entirely.

```bash
opencode run \
  --dir "<absolute workdir>" \
  -m <provider/model> \
  --format json \
  --title "<short session name>" \
  < brief.txt > "<logfile>" 2>&1
```

PowerShell has no `<` input redirection, so pipe there instead:
`Get-Content brief.txt -Raw | opencode run --dir ... --format json`

| Flag | Why you'd use it |
|---|---|
| `--dir` | Run in the target repo without `cd`. |
| `-m provider/model` | The confirmed worker model. |
| `--format json` | Structured JSONL instead of a formatted transcript. |
| `--title` | Names the session so `opencode session list` can find it. |
| `--agent` | Pick an agent profile - see read-only below. |
| `-c` / `-s <id>` | Continue the last / a specific session, for corrections. |
| `-f <file>` | Attach files to the message. |
| `--variant` | Provider-specific reasoning effort (`high`, `max`, `minimal`). |

Keep the brief in a file even when it is short. A file is what you re-send on a
retry, what you show the user when they ask what was sent, and what makes the
run reproducible.

**Run it in the background and redirect to a log.** Implementation runs
routinely take an hour or more, and local models are slower still; a foreground
call that hits a tool timeout loses the run and tells you nothing about why. Set
a time limit you're willing to wait, and know that a killed or timed-out run can
leave **partial edits** - inspect the tree before retrying or cleaning, or you
will retry on top of half-finished work.

**Read-only delegation.** For a review or diagnosis that must not touch the
tree, dispatch with an agent whose permissions forbid edits - conventionally
`--agent plan`. Check the profile rather than trusting the name: agents come
from local config, so `plan` may not exist, and its name alone guarantees
nothing. `opencode agent list` shows each agent's actual permissions.

**Unattended mode and `--auto`.** `--auto` auto-approves every permission not
explicitly denied. That is the point of an unattended run - a headless worker
can otherwise block on a prompt nobody can answer - and it is also a real
escalation of trust, because the safety net becomes your review rather than a
per-action gate. It is acceptable when all of these hold:

- The repository is trusted and the working directory is scoped to the task.
- The tree is clean, or the pre-existing changes are recorded.
- The brief authorizes no destructive, network, or install actions.
- The user has approved it for this run, and you note that in the report.

Otherwise prefer scoping permissions in the opencode config. Don't reach for
`--auto` merely because a run stalled on a prompt.

If the user already has an interactive opencode TUI open and wants the work to
land there, attach to it (`opencode attach <url>`) rather than starting a
competing session. Leave their terminal running - killing it is theirs to decide.

## 5. Monitor for the result, not the process

Read the capture for the final deliverable and for what needs your intervention:
permission requests, hard errors, an attempt to touch something out of scope, a
stall. Thinking blocks and routine tool chatter are not signals - reading them
burns your context and tells you nothing you'll act on. One line of status is
plenty if the user asks. When a run misbehaves - empty result, unknown model, a
hang, a timeout, partial edits - see `references/recovery.md`.

## 6. Verify the result

Judge the worker's final output against the user's original request. The
dominant failure mode of a local worker, by a wide margin, is a confident
summary of work that never happened, so the first question is always whether the
claims match reality.

- Did it complete the task, or describe completing it? Check the filesystem.
  A worker reporting success is not evidence of success.
- Did it work in the intended directory, with the intended model?
- Did it stay inside the file, command, and permission boundaries?
- Did it produce the evidence or format that was asked for?
- Do its claims contradict what's on disk?

**If the task changed code or a working tree**, add diff review, the project's
own check commands, and the commit boundary - `references/code-tasks.md`.

**If it didn't**, the equivalent is verifying the artifacts against the stated
success criteria: the files exist, contain what was asked, and are internally
consistent. Don't invent gates a task never had.

## 7. Retry or report

When a retry is warranted, continue the same session (`-c` or `-s <id>`) so the
worker keeps its context, and send a delta - what was wrong, what to fix, what to
report back - rather than restating the whole task. Don't bundle in unrelated
improvements; that's how a near-miss becomes a fresh mess.

Scale retries to the stakes. Two or three focused attempts is usually the honest
limit. If the worker isn't converging, that is a real finding about the model,
and the user would rather hear it than watch you grind.

Running several tasks in a row has its own discipline - see
`references/brief-template.md`.

## Permissions and safety

- Keep the user's opencode terminals and sessions alive unless asked to stop them.
- Don't approve destructive cleanup, edits outside the working directory,
  installs, or network actions unless the user authorized that class of action.
  If the worker asks for one, pause and ask rather than deciding for them.
- If the worker wrote to the wrong place, have it recover inside the authorized
  workspace. Don't delete external paths on your own initiative.

## Final report

Label who did what. The user is evaluating a model; blurring authorship destroys
the only thing the exercise was for. Use these headings:

- **Worker result** - what the worker produced, as the worker's.
- **Supervisor verification** - what you checked and found. Files changed, commands
  run, artifacts produced: verified, not merely claimed.
- **Caveats** - anything you fixed yourself, anything unverified, anything still
  needing a user decision.

Include the model and session id so the run can be resumed, the verdict
(accepted / accepted with caveats / retried / blocked), and whether `--auto` was
used. Surface rather than absorb: report the worker's design decisions,
defensible but unasked turns, and nitpicks instead of silently keeping them.

## Command reference

```bash
opencode models [provider]   # available models
opencode agent list          # agent profiles and their permissions
opencode session list        # sessions, with ids
opencode export <sessionID>  # session transcript as JSON
opencode stats               # token usage and cost
```

## References

- `references/brief-template.md` - the brief structure, a reusable template, and
  running a queue of tasks. Read before writing the first brief.
- `references/code-tasks.md` - diff review, project gates, reviewing test edits,
  and the commit boundary. Read when the delegated task touches a git tree.
- `references/recovery.md` - what to do when a run comes back empty, hangs,
  times out, or leaves partial edits. Read when a run misbehaves.
