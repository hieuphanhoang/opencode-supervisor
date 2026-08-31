# opencode-supervisor

An Agent Skill for delegating a task to a model running under the
[opencode](https://opencode.ai) CLI and supervising it end to end: choose a
suitable worker model, write a brief the worker can execute blind, dispatch it
headlessly, verify what came back against what was actually asked, retry
narrowly when it falls short, and report with authorship kept straight.

Works with any agent harness that can run a shell command and read a file -
Claude Code, Codex CLI, Cursor, and others. The skill is provider-neutral about
who supervises; opencode is the harness being supervised.

## Why this exists

Handing work to another model is easy. Knowing whether it actually did the work
is not.

The failure this skill is built around is specific and common, especially with
local models: a confident final report describing work that never happened.
Files that were never written, tests that were never run, a summary that reads
well and cannot be reconciled with the disk. A supervisor that trusts the
worker's self-report catches none of it.

So the loop here is deliberately asymmetric. The worker does the work. The
supervisor verifies it against the filesystem, not against the worker's account
of itself, and reports the two separately so the user can always tell which is
which.

## What it covers

- **Preflight** - is opencode installed, which agents and models does this
  install actually reach, is the local provider up. Shell-neutral, with POSIX
  and PowerShell forms where they differ.
- **Model choice** - hard gates (reachable, tool-calling, enough context) before
  ranking, with fit separated from rank because a benchmark task names its own
  model. The user can choose, state a standing policy, or delegate the choice -
  and delegated choice never reaches a metered model unless the policy says so.
- **Briefing** - a template for instructions a worker can follow with no other
  context, and the delivery path that does not get mangled: the brief goes in on
  stdin, never as a shell argument.
- **Dispatch** - structured JSONL capture rather than scraping an ANSI
  transcript, read-only delegation via a plan-style agent, and an explicit
  unattended mode setting out when auto-approving permissions is acceptable.
- **Verification** - check the claims against the disk. For code tasks, review
  the diff, re-run the project's real gates, and review test edits before
  trusting a green run.
- **Recovery** - empty results, unknown models, permission hangs, timeouts that
  leave partial edits.

## Installing

With the [skills.sh](https://www.skills.sh) CLI:

```bash
npx skills add hieuphanhoang/opencode-supervisor
```

It detects the agents you have installed and writes the skill into each one's
skills directory, so this is the same command whether you run Claude Code,
Codex CLI, Cursor, or several at once.

To install by hand instead, copy this folder into your agent's skills
directory:

- Claude Code: `~/.claude/skills/opencode-supervisor/`
- Codex CLI: `~/.codex/skills/opencode-supervisor/`
- Tools following the `AGENTS.md` convention: `~/.agents/skills/opencode-supervisor/`

Keep the folder intact - `SKILL.md` links to the files under `references/`, and
`agents/openai.yaml` supplies the display name Codex shows.

The skill triggers on requests like "have opencode do X", "delegate this to a
local model", "run this through opencode", or "pick the best model for this
task".

## Requirements

- The [opencode](https://opencode.ai) CLI, installed and able to reach at least
  one provider - cloud credentials or a running local provider such as Ollama.
- A supervising agent that can run shell commands and read files.

## Layout

```
opencode-supervisor/
  SKILL.md                      the loop: preflight, model choice, brief,
                                dispatch, verify, retry, report
  references/
    brief-template.md           writing a brief, and running a queue of tasks
    code-tasks.md               diff review, gates, and the commit boundary
    recovery.md                 when a run comes back empty, hangs, or times out
```

`SKILL.md` holds the decisions needed on every run; the references are consulted
when the situation calls for them.

## Licence

MIT - see [LICENSE](LICENSE).
