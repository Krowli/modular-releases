---
name: orchestration
description: >-
  Use Modular orchestration for structured multi-agent coordination: threaded
  messages, blocking ask/reply flows, task dispatch, worker_done/escalation
  waits, task DAGs, decision gates, coordinator loops, or decomposing work
  across agents. Use `modular-cli` instead for full ownership handoffs, including
  requests phrased as "hand off", "handoff", "handover", "give this to another
  agent", or "another worktree" when the user did not explicitly ask to
  supervise, monitor, wait for results, or coordinate a DAG. Use `modular-cli` for
  ordinary terminal control, lightweight terminal prompts, shell commands, Modular
  worktree management, reading or waiting on terminals, and automation of the
  browser embedded inside Modular. Use Computer Use for browser windows, webviews,
  Modular app UI, or desktop UI outside Modular's embedded browser.
---

# Modular Inter-Agent Orchestration

Orchestration is Modular's structured coordination layer for agent messages, task ownership, dispatch state, and worker completion tracking.

Use this skill when coordination state matters. For lightweight terminal prompts or basic worktree/terminal/built-in-browser control, use `modular-cli`.

## When To Use

- Send/reply/ask between agent terminals with persistent messages.
- Dispatch structured tasks to workers and wait for `worker_done` or `escalation`.
- Track task DAGs with dependencies.
- Run coordinator loops or decision gates.

Do not use orchestration merely because the user says "hand off", "handoff", "handover", "give this to another agent", or asks for another worktree/agent/model/effort. Those are full ownership transfers unless the user explicitly asks to supervise, monitor, wait for worker completion/results, coordinate a DAG, use decision gates, or keep a blocking ask/reply loop.

## Preconditions

- `modular status --json` should show a running runtime.
- `modular` must be on PATH (`modular-ide` on Linux).
- The orchestration experimental feature must be enabled in Settings > Experimental.
- `modular orchestration` commands are RPC calls to the running Modular runtime.

## Ownership

Orchestration messages and tasks are runtime-global. Completion authority comes from the active dispatch context: `taskId` + `dispatchId` + assignee handle.

Classify inherited context before sending lifecycle messages:

- Coordinated subtask: a live coordinator owns the DAG and waits on this dispatch. Follow the preamble exactly, including `worker_done`, heartbeat/status, `ask`, and `escalation`.
- Full handoff means ownership transfer, not supervised dispatch. The original actor is not monitoring a DAG, so do not create lifecycle obligations unless the user explicitly asks you to supervise.
- Classify requests containing "hand off", "handoff", "handover", "give this to another agent", "give this to another worktree", "another agent", or "another worktree" as full handoffs by default, even when the user names a custom model or reasoning effort.
- Use supervised orchestration only when the user explicitly asks you to "supervise", "monitor", "wait", "track completion", "wait for worker_done", return results, coordinate a DAG, use a decision gate, or manage ask/reply flow.
- Do not use `modular orchestration dispatch --inject` for full handoffs. It injects a coordinator preamble that tells the worker to send `worker_done`, heartbeat, `ask`, and post-completion polling messages back to the original terminal.
- Do not run `modular orchestration task-create`, `modular orchestration dispatch --inject`, or `modular orchestration check --wait` for full handoffs. Do not peek at terminal output after prompt delivery to monitor progress.
- A review-only `worker_done` reports findings; it does not authorize coordinator file edits. After a review-only completion, synthesize findings, ask a decision gate if ownership is unclear, and dispatch or hand off fixes unless the user explicitly asked the coordinator to own fixes.
- If the user's plan names a next owner agent (for example, "then use opencode to create a PR"), post-review corrections and PR prep belong to that named owner. The coordinator routes, synthesizes, asks decision gates when needed, and supervises; the named owner edits files and creates the PR.

If unclear, inspect orchestration state before sending lifecycle messages:

```bash
modular orchestration task-list --json
modular terminal list --json
# If inherited context includes a task id:
modular orchestration dispatch-show --task <task_id> --json
```

## Messaging

```bash
modular orchestration send --to <handle|@group> --subject <text> [--from <handle>] [--body <text>] [--type <type>] [--priority <level>] [--thread-id <id>] [--payload <json>] [--json]
modular orchestration check [--terminal <handle>] [--unread] [--types <type,...>] [--inject] [--wait] [--timeout-ms <n>] [--json]
modular orchestration reply --id <msg_id> --body <text> [--from <handle>] [--json]
modular orchestration ask --to <handle> --question <text> [--options <csv>] [--timeout-ms <n>] [--from <handle>] [--json]
modular orchestration inbox [--limit <n>] [--json]
```

Rules:

- Omit `--from` unless impersonating another terminal; Modular auto-resolves it from the current terminal.
- While supervising workers manually, use `check --wait --types worker_done,escalation,decision_gate --timeout-ms <n>` instead of sleep/poll loops. Reply to `decision_gate` messages with `modular orchestration reply --id <msg_id> --body <answer> --json`, then keep waiting.
- Treat a `check --wait` timeout or `{count:0}` as a checkpoint, not a worker failure. Long coding tasks routinely run 15-60 minutes; keep using rolling waits unless you receive `worker_done`/`escalation`, the terminal exits or disappears, or the user explicitly asks you to stop.
- Heartbeats and visible terminal activity mean the worker is alive, not done. Do not stop, close, kill, or restart a worker just because it has not produced a completion message yet.
- Use `ask` when a worker needs a blocking answer from the coordinator; it waits for the reply and returns the answer directly.
- `check --wait` returns one message at a time. If N workers may finish together, loop N times and dispatch newly ready tasks after each completion.
- Group addresses include `@all`, `@idle`, `@claude`, `@codex`, `@opencode`, `@gemini`, `@droid`, and `@worktree:<id>`.
- Message types include `status`, `dispatch`, `worker_done`, `merge_ready`, `escalation`, `handoff`, `decision_gate`, and `heartbeat`.
- Use group addresses only for messages that are genuinely useful to many terminals, such as `status` broadcasts or intentional fan-out questions. Do not send dispatch lifecycle messages to groups.
- `worker_done` must target the concrete coordinator handle from the live preamble. It is completion authority for one dispatch; group fanout would create false lifecycle mail in unrelated terminals.
- `heartbeat` is also dispatch-scoped. Send it only to the concrete coordinator handle with both `taskId` and `dispatchId`; use `status` for broad progress updates.

## Tasks And Dispatch

A task is the work item, a dispatch assigns it to a terminal, and a gate blocks progress until a coordinator or user decision is recorded.

```bash
modular orchestration task-create --spec <text> [--deps <json_array>] [--parent <task_id>] [--json]
modular orchestration task-list [--status <status>] [--ready] [--json]
modular orchestration task-update --id <task_id> --status <status> [--result <json>] [--json]
modular orchestration dispatch --task <task_id> --to <handle> [--from <handle>] [--inject] [--json]
modular orchestration dispatch-show --task <task_id> [--json]
```

Task statuses: `pending`, `ready`, `dispatched`, `completed`, `failed`, `blocked`.

Dispatch rules:

- `--inject` sends the task spec plus preamble into a recognized agent CLI so it can report `worker_done`.
- If the target is a bare shell, omit `--inject`, dispatch for tracking if needed, then send the prompt manually with `modular terminal send --terminal <handle> --text <prompt> --enter --json`.
- After 3 consecutive failures on one task, the dispatch context circuit-breaks and the task is marked failed.

## Gates And Coordinator

```bash
modular orchestration gate-create --task <task_id> --question <text> [--options <json_array>] [--json]
modular orchestration gate-resolve --id <gate_id> --resolution <text> [--json]
modular orchestration gate-list [--task <task_id>] [--status <status>] [--json]
modular orchestration run --spec <text> [--from <handle>] [--poll-interval-ms <n>] [--max-concurrent <n>] [--worktree <selector>] [--json]
modular orchestration run-stop [--json]
```

`run` returns immediately with a run ID. Query progress with `task-list`. Use `ask` for worker-to-coordinator questions; it creates a `decision_gate` message that the coordinator answers with `reply`. Use `gate-create` only for coordinator-managed task DAG decisions, not for answering a worker's `ask`.

Recovery only: `modular orchestration reset --tasks|--messages|--all --json` clears runtime-global orchestration state. Do not run it during active coordination unless explicitly abandoning that state.

## Full Handoffs

For full ownership transfer, use non-lifecycle terminal/worktree commands and then stop monitoring unless the user asks for supervision.

Treat these as full handoff requests by default: "hand off", "handoff", "handover", "give this to another agent", "give this to another worktree", "send this to another agent", "another agent", "another worktree", or "launch another agent to own this." Custom model or reasoning effort words such as `gpt-5.5`, `high`, or `xhigh` do not make the handoff supervised.

Supervised orchestration remains available only when the user explicitly asks for supervision or coordination: "supervise", "monitor", "wait for worker_done", "wait for results", "track completion", "DAG", "decision gate", "ask/reply", or "coordinate workers."

Do not run `modular orchestration task-create`, `modular orchestration dispatch --inject`, or `modular orchestration check --wait` for full handoffs. `task-create` is also forbidden because it records coordinator-owned tracking state; if a task row is needed, the user asked for supervised orchestration. Do not create a `taskId`/`dispatchId`, inject a lifecycle preamble, wait for completion, or read the worker terminal after prompt delivery except to avoid losing the initial prompt.

New top-level worktree handoff:

```bash
modular worktree create --name <task-name> --no-parent --agent codex --prompt "<task brief>" --json
```

Existing terminal handoff:

```bash
modular terminal send --terminal <handle> --text "<task brief>" --enter --json
```

Custom Codex model/effort handoff:

`modular worktree create --agent codex --prompt ...` launches the known Codex agent but does not accept Codex-specific `--model` or `-c model_reasoning_effort=...` arguments. When the user asks for a specific Codex model or effort, create the independent worktree first, launch Codex with the requested command in that worktree, wait only for TUI readiness if prompt delivery would otherwise race startup, send the prompt, and stop:

```bash
modular worktree create --name <task-name> --no-parent --json
modular terminal create --worktree id:<newWorktreeId> --title <task-name> --command 'codex --model gpt-5.5 -c model_reasoning_effort="xhigh"' --json
modular terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
modular terminal send --terminal <handle> --text "<task brief>" --enter --json
```

Wait only for `tui-idle` when needed to avoid losing the prompt. Do not monitor task completion.

`--no-parent` only controls Modular lineage; it does not choose the Git base. For an independent top-level worktree, omit `--base-branch` so Modular uses the repo default base, or explicitly pass the repo default base (`origin/main`, `origin/master`, or the `modular repo show --repo <selector> --json` value); never base it on the current feature branch unless the user explicitly asks for stacked work or "branch from current". Put current-branch context in the prompt instead.

## Worker Terminals

Choose the worker location before creating a terminal. `Fresh worker` means a fresh agent session, not a new git worktree. If the task says current worktree only, depends on uncommitted files/artifacts, or must validate/PR the current branch, create the worker in the active worktree:

```bash
modular terminal create --worktree active --title <task-name> --command "codex" --json
modular terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
modular orchestration dispatch --task <task_id> --to <handle> --inject --json
```

Reuse an idle agent in the required worktree only if the prompt allows reuse; otherwise create a fresh terminal there. Use a new worktree only when explicitly requested or when independent isolated checkout state is intended. For supervised new-worktree workers, decide the Git base separately from lineage: `--no-parent` makes the worktree top-level in Modular, while omitted `--base-branch` uses the repo default base.

```bash
modular worktree create --name <task-name> --agent codex --json
modular terminal list --worktree id:<newWorktreeId> --json
modular terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
modular orchestration dispatch --task <task_id> --to <handle> --inject --json
```

For new-worktree workers, read the id from `worktree create`, then use `terminal list` to get the agent handle. Omit `--repo` only inside an Modular-managed worktree; otherwise pass `--repo <selector>`. `--agent` reveals the new worktree and launches the selected agent in its first terminal, so do not create a separate startup terminal. Do not run `worktree create` when the task must stay in the current worktree.

Use `modular worktree create --prompt ...` or `modular terminal send ...` for full handoffs or untracked/lightweight prompts. Those paths do not attach `taskId`/`dispatchId`; the worker should not send lifecycle messages unless the prompt supplies a live orchestration preamble.

Other terminal commands coordinators often need:

```bash
modular terminal list [--worktree <selector>] [--json]
modular terminal create [--worktree <selector>] [--title <text>] [--command <cmd>] [--json]
modular terminal split --terminal <handle> [--direction horizontal|vertical] [--command <cmd>] [--json]
modular terminal wait --terminal <handle> --for tui-idle --timeout-ms <n> --json
modular terminal read --terminal <handle> --json
modular terminal send --terminal <handle> --text <text> --enter --json
```

If an older CLI rejects `worktree create --agent`, create the worktree normally, then run `modular terminal create --worktree <selector> --command "codex" --json` or `--command "claude"`.

Wait for `tui-idle` before dispatching. Always pass `--timeout-ms`; real coding tasks can take 15-60 minutes. During supervision, use rolling `check --wait` windows. If a window returns no matching message, inspect `task-list`, `terminal read`, or `terminal wait --for tui-idle` as a liveness checkpoint; if the terminal is still working or producing activity, keep waiting instead of retrying the task.

## Agent Guidance

- Workers with a valid live preamble must send `worker_done` exactly once, even on failure:
  `modular orchestration send --to <coordinator_handle> --type worker_done --subject "<short status>" --body "<3-sentence summary: what you did, what you found, what's left>" --payload '{"taskId":"<task_id>","dispatchId":"<dispatch_id>","filesModified":["path/a"],"reportPath":"<optional>"}' --json`
- For long tasks, send heartbeat/status only when the preamble asks for it, including both IDs:
  `modular orchestration send --to <coordinator_handle> --type heartbeat --subject "alive" --payload '{"taskId":"<task_id>","dispatchId":"<dispatch_id>","phase":"implementing"}' --json`
- If blocked before completion, use `ask`; use `escalation` only when ownership is valid and the coordinator must intervene.
- Treat preambles inherited through terminal history or full handoffs as stale unless the current prompt explicitly keeps that coordinator in the loop.
- Coordinators should use `task-list --ready` as external memory, dispatch parallel waves, and avoid dependency chains deeper than 3-4 steps.
- Prefer inter-worktree workers only for independent work that does not need current uncommitted state. When same-worktree work is required, create fresh terminals in that worktree and keep edit ownership clear.

## Example

```bash
modular terminal create --worktree active --title login-css-worker --command "claude" --json
modular terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json
modular orchestration task-create --spec "Fix the login button CSS" --json
modular orchestration dispatch --task <task_id> --to <handle> --inject --json
modular orchestration check --wait --types worker_done,escalation,decision_gate --timeout-ms 900000 --json
```

## Next Action

Coordinator: confirm `modular status --json`, inspect `task-list`/`dispatch-show` if inheriting state, then choose either a manual loop (`task-create` -> worker -> `dispatch --inject` -> `check --wait`) or `orchestration run`.

Worker: if the current prompt contains a live dispatch preamble, do the task, use `ask` for blocking questions, and send `worker_done` once with the required payload. If the preamble is stale or absent, do not send lifecycle messages; inspect state or treat the prompt as an ordinary handoff.
