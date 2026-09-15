# kanban: a worker that calls `delegate_task` wedges its card indefinitely, and the heartbeat hides it

## Summary

A kanban worker that decides to delegate its work to a sub-agent enters a state the board cannot see: the
parent blocks waiting for a result its child can never turn into a terminal kanban call, the run keeps
**heartbeating**, and the card therefore stays `running` forever while producing nothing. Observed live:
**1 h 45 m of silence on one card, no commits, no file changes, after two earlier runs on the same card were
reaped as `killed_stuck_25min`.**

## Observed

- Task: `t_397c853f` (board `hermes-console`, assignee `coder`, workspace `worktree`, branch `wt/ws-recovery`).
- Runs table: `#6 killed_stuck_25min` (1729 s) → `#7 killed_stuck_25min` (1946 s) → `#8 running` (reclaimed
  by hand at 6943 s).
- The worker's log freezes immediately after:
  `Planning task delegation with kanban subtasks` → `preparing delegate_task…` — and nothing further.
- The worker process is alive with **no child processes**, and the two files the card names are **never
  modified** (`git log origin/main..HEAD` empty; file mtimes unchanged).
- The delegated child's own record, recovered from the worker's session DB, reports:
  `api_calls: 86`, `duration_seconds: 6248`, `exit_reason: interrupted`.

## Why it matters

- ~3 hours of worker time and a stalled deliverable, with **no alert anywhere**: watchdogs that only look at
  terminal states (`done`/`blocked`) are blind, and `killed_stuck` rows are the only trace.
- A `running` card looks healthy to every status surface, so a human watching the board is actively misled.

## Expected behaviour (any of these would be enough)

1. `delegate_task` is **refused inside a kanban worker run** (children cannot make kanban mutations, so the
   pattern is always a dead end), or
2. the harness detects a worker whose log/commits have not moved for N minutes **while heartbeating** and
   transitions the card to `blocked` with a precise reason, or
3. `killed_stuck_*` triggers a visible alert and the card is not silently re-dispatched with the same body.

## Suggested minimal fix

- Inject a "do not delegate; commit after each file" clause into every generated card body (a comment is not
  enough — a running worker never reads comments).
- Add a heartbeat-with-no-progress detector: if `running` AND log size unchanged AND no new commits for
  > 25 min → `blocked` with reason `no_progress_wedge`.

## Environment

- Hermes Agent, gateway **0.21.2**, stock install (checkout `~/workspace/hermes-agent-stock`).
- Host: WSL2 (Ubuntu 24.04) on Windows 10; board DB `~/.hermes/kanban/boards/hermes-console/kanban.db`.
- Related earlier sighting of the same class: 2026-08-15 delegate-deadlock on `t_7b2386b4`.
