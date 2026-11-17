---
name: humanize
description: Run a coding or research task through humanize (hmz), an agent-flow orchestrator that drives Claude Code, Codex and other agent CLIs in loops such as actor + reviewer (rlar). Use when the user says "use humanize", "use hmz", "run it in hmz", "rlar this", or wants a task done by an implementer agent checked by a separate reviewer agent, a long unattended agent loop (hours to days), or Claude implementing while Codex reviews. Not for rewriting text to sound more human.
---

# humanize (hmz)

humanize (`hmz`, https://github.com/humanfia/humanize, docs https://docs.humanfia.ai/humanize/)
runs coding-agent CLIs (`claude`, `codex`, `agy`, …) inside **flows**: loops that decide who
works, who reviews, and when the run stops. It is a uv tool from PyPI (`uv tool install hmz`,
`~/.local/bin/hmz`); `hmz update` upgrades it. Bare `hmz` opens a TUI for the current
directory, `hmz exec` runs a flow headless, `hmz web` serves every workspace's runs in a
browser. Written against 0.2.2; this alpha changes often, so trust `hmz exec --help` over
this file.

## The user's defaults (edit for your setup)

| Setting | Default |
|---|---|
| Flow | `rlar`: Claude implements, Codex reviews |
| Actor | `claude/claude-opus-5-5:high` |
| Reviewer | `codex/gpt-6.2-sol:high` (sometimes `:ultra`) |
| Budget | `budget.duration=1d`, with no cost or token cap |

Use these unless the user says otherwise, or the workspace has saved defaults (below).
Don't ask about the budget.

```sh
hmz exec -f rlar \
  -a actor=claude/claude-opus-5-5:high \
  -a reviewer=codex/gpt-6.2-sol:high \
  -p budget.duration=1d \
  --json "$(cat "$TASK")" > "$LOG" 2>&1
```

## Workflow

1. **Pre-flight.**
   - Look up the directory in `~/.hmz/settings.yaml` under `workspaces:`. If it has an entry,
     prefer its flow, agents, envs and budget over the defaults above.
   - Check every model and effort you pass exists in `~/.hmz/models/<cli>.json`. hmz doesn't
     validate specs up front: a bad one fails only on the first turn that uses it, so a wrong
     reviewer model in `rlar` surfaces after the actor's whole first turn.
   - Check `git status`. hmz agents run with approvals bypassed. If the repo has uncommitted
     work you didn't create, or it can change live systems (infrastructure, deploy scripts,
     credentials), say so and suggest a git worktree before starting.
2. **Write the task file** (next section) in the scratchpad and show it to the user if the
   task is non-trivial or was given loosely.
3. **Launch in the background** with `run_in_background` and `--json` redirected to a log in
   the scratchpad. Runs last hours to days. Report the command, the task file, the log path,
   and how to watch it: `hmz` in that directory opens the TUI on the run, or `hmz web`.
4. **Read the result** (see Results), then check the repo diff and run the project's checks
   yourself before calling it done. The reviewer's "done" is evidence, not proof.
5. **Stopping.** Stop the background task you started. Never bare `pkill hmz`, since the user
   often has other runs going. Inspect `pgrep -af hmz` and stop only your PID.
6. **Resuming.** `--resume` with the same `-f` in the same directory (limits below).

## Writing the task

The actor only sees the task, and in `rlar` the reviewer judges the repo against it, so
vague tasks give endless review rounds or a premature "done". Include:

- **Goal**: the outcome in one or two sentences, not the steps.
- **Read first**: the files, specs or docs that define the work (e.g. an OpenSpec change,
  an issue, a design note), so the agent doesn't rediscover them.
- **Done when**: concrete acceptance criteria the reviewer can check: commands that must
  pass, behavior that must hold, numbers to hit.
- **Out of scope**: what not to touch, and any action that needs a human (deploys, pushes,
  deletions, paid services, live-system changes).
- **Conventions**: commit policy (commit or leave uncommitted, message style), where to put
  logs and artifacts.

Write it to a file and pass `"$(cat "$TASK")"`, or tell the agent to read the file first.
A task starting with `-` goes after `--`.

## Command shape

`hmz exec -f FLOW -a ROLE=SPEC[,...] [-e ROLE=SPEC[,...]] -p KEY=VALUE[,...] [opts] task`

- `-f`: a flow name. Built-ins are bare (`rlar`); the user's own are `@user/NAME` (from
  `~/.hmz/flows/NAME/`) and the project's `@local/NAME`. Also a path starting with `.`, `/`
  or `~`, `git+URL#subdir`, or `NAME:OTHER` for another flow in the same module. A bare name
  resolves nearest first, so a user flow can shadow a built-in.
- `-a`: `ROLE=CLI[@PROVIDER]/MODEL[:EFFORT]`, one per agent role the flow defines. Repeat the
  flag or comma-separate.
- `-e`: `ROLE=BACKEND[@PROVIDER][/WORKDIR]` for flows with environment roles. `PROVIDER` is a
  runtime saved in settings or, for ssh only, a host in brackets:
  `box=ssh@[user@host:22]/abs/path`. Without a provider it is this machine.
- `-p`: flow params, parsed as the param's type or as JSON. Every flow except `chat` needs a
  budget: `budget.duration` (`1d`, `2h30m`, `PT1H`), `budget.cost` (USD),
  `budget.output_tokens`. `budget.graceful=true` lets the last turn finish (or
  `budget.graceful.<limit>` for one limit).
- `--json`: NDJSON events to stdout. Use it when you need to read the result.
- `--resume`: picks up the newest run of this flow here, if the flow can be resumed
  (`ralph_loop` and `continue_loop` can't). Resuming `rlar` hands a fresh actor only the last
  review, not the old actor session.
- `--backup NAME`: back the epic up to a backup target saved in settings (`none` turns it off).
- `--profile-flow` / `--profile-environment`: add sampling to the run's trace. Use them only
  when diagnosing a slow run.
- A failed turn ends the run in every built-in loop flow, so don't count on a loop retrying it.

## Choosing a flow

| Flow | Roles | Use when |
|---|---|---|
| `rlar` | `actor`, `reviewer` | Default. Work that needs a separate check. The actor works in one session, a fresh reviewer checks the repo against the actor's claims each round, and it ends when the reviewer says done. |
| `chat` | `assistant` | One agent, one shot, no loop and no budget. Questions and small edits. |
| `goal` | `worker` | One `/goal <task>` session that runs until the model says it's met. Self-checking work with clear criteria. |
| `continue_loop` | `agent` | Sends the task once, then keeps sending "continue". Long grinds in one context. |
| `ralph_loop` | `agent` | Fresh session every round with the same task. Progress lives in the repo, not the context. |
| `stateful_ralph` | `agent` | One session, task re-sent every round. |
| `flame_chase` | `first_chaser`, `second_chaser` | Two agents take turns in fresh sessions. |

User flows live in `~/.hmz/flows/<name>/__init__.py`. List them with `ls ~/.hmz/flows/` and
read the docstring for roles and params before using one. To get any flow's full contract:
`~/.local/share/uv/tools/hmz/bin/python -c "import hmz.flows.builtin.rlar as m; print(m.__doc__)"`.

## Results

- Every run is an epic under `~/.hmz/epics/<cwd-with-slashes-as-dashes>/<timestamp>/`, and
  `epic.jsonl` holds the summary events: `began` (flow, task), `opened` (each agent session),
  `usage` (cost, output_tokens, seconds), and `ended` with a `how` field: `done` when the
  flow's own stop condition was met, `stopped` otherwise (other values may exist).
- The `--json` log has the full event stream, including errors such as
  `HarnessUnrecoverable` when a turn fails. Read its tail for why the run ended.
- Report the outcome, the cost and duration from `usage`, and your own diff check.

## Settings notes

- Saved runtimes say where the agent CLI runs with `harness:` (`affinity:` is the old name
  and still read). A runtime on another machine with no harness set doesn't fall back to
  running the CLI here, so set `harness: local` when the remote host doesn't have the CLI.
- `/btw` side questions to the epic only work once the `btw` setting names an agent
  (Settings › General).
- `hmz web` run pages are under `#/epics/`.
