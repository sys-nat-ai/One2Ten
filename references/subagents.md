# Subagents Reference — Build-phase orchestration

You are the orchestrator: dispatch, review, decide. Never implement
directly — a manual "quick fix" pollutes your context and skips review.
Narrate at most one short line between dispatches.

Run tasks sequentially by default. Parallel dispatch is allowed only for
independent tasks on disjoint systems (e.g., a Supabase migration and an
unrelated Make scenario).

## Implementer dispatch (fresh subagent per task)

Before dispatching: extract the task's full text from the plan to
`.one2ten/briefs/task-N-brief.md`. For code tasks, record the BASE commit
(`git rev-parse HEAD`) — you need it for review.

Dispatch prompt skeleton:

```
<one line: where this task fits in the ticket>

Read .one2ten/briefs/task-N-brief.md first — it is your requirements, with
the exact values to use verbatim.

Context from earlier tasks: <interfaces/decisions the brief cannot know>

Safe-change rules: fetch current state before editing anything live; prefer
additive changes; validate before publishing; never deactivate/delete
production assets.

Write your full report to .one2ten/briefs/task-N-report.md (what you did,
tool calls/commands run, their observed outputs). Return ONLY: a status,
one-line summary, and any concerns.

Status vocabulary:
- DONE — task complete, verify step passed with evidence in the report
- DONE_WITH_CONCERNS — complete, but read my concerns
- NEEDS_CONTEXT — missing information, ask and I will re-dispatch
- BLOCKED — cannot complete; reason in report
```

## Status handling

- **DONE** → dispatch the reviewer.
- **DONE_WITH_CONCERNS** → read the concerns first. Correctness/scope
  concerns get addressed before review; observations get noted.
- **NEEDS_CONTEXT** → answer completely, re-dispatch.
- **BLOCKED** → change something: more context, a more capable model, or
  split the task. If the plan itself is wrong, stop and tell the human.
  Never silently retry the same dispatch.

## Reviewer dispatch (after every task)

The reviewer gets: the brief path, the report path, and the change
evidence. It must return TWO verdicts — **spec compliance** (everything in
the brief done, nothing extra) and **quality** — plus findings rated
Critical / Important / Minor.

Change evidence by work type:
- **Code:** write `git diff BASE..HEAD` (BASE recorded at dispatch — never
  `HEAD~1`, it drops earlier commits of a multi-commit task) plus
  `git log --oneline BASE..HEAD` to `.one2ten/briefs/task-N-diff.txt`;
  give the reviewer that path.
- **MCP work (no git diff exists):** instruct the reviewer to fetch live
  state itself and check it against the brief — n8n `get_workflow_details`,
  Make `scenarios_get` (blueprint), Supabase `list_migrations` /
  `list_tables` — and to check the implementer's execution evidence in the
  report.

Never tell a reviewer what not to flag, and never pre-rate a finding's
severity in the dispatch.

## Fix loop

- Critical/Important findings → dispatch a fix subagent with the complete
  findings list (one fixer, not one per finding); it appends its fix report
  with re-run verification to the same report file → re-dispatch the
  reviewer. Repeat until clean.
- Minor findings → record in the ledger; the final reviewer triages them.
- Never move to the next task with open Critical/Important findings.

## Progress ledger — `.one2ten/progress.md`

Append one line when a task's review comes back clean:

```
Task N: complete (commits <base7>..<head7> | n8n workflow <id> updated+published, review clean)
```

After compaction or resume: trust the ledger plus `git log`/live MCP state
over your own memory. Never re-run a task the ledger marks complete.

## Model selection (specify on every dispatch)

- Plan contains the complete content (transcription + verification) →
  cheapest tier (haiku).
- Integration work, multi-file or multi-node coordination → standard tier
  (sonnet).
- Final whole-change review → most capable available model.
- Task reviewers → scale to the change's size and risk.
