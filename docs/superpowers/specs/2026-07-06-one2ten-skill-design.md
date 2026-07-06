# One2Ten Skill — Design

**Date:** 2026-07-06
**Owner:** Nat (AI Engineer)
**Status:** Approved design, pending spec review

## What this is

A personal Claude Code skill, `/one2ten`, that codifies Nat's end-to-end
process for handling task tickets: implementing new features, doing tasks,
and fixing bugs. Work spans n8n workflows, Make scenarios, Supabase, and
regular code/apps/voice agents, driven through MCPs.

The flow is **Understand → Plan → Build → Test → Report**, with one human
gate (plan approval). Planning absorbs the superpowers `writing-plans`
method; Build absorbs `subagent-driven-development` — fresh subagents
implement each task, reviewer subagents verify each one.

## Where it lives

- **Installed:** `C:\Users\Nat Pisco\.claude\skills\one2ten\` — available in
  every project as `/one2ten`.
- **Developed:** this repo (`One2Ten`), which holds the source of truth.
  After edits here, copy to the install location.

## Inputs

Tickets arrive two ways:

1. **Zoho Projects** — a task or issue reference. The skill fetches details
   via the Zoho Projects MCP (`get_task_details`, `get_issue`, etc.).
2. **Chat briefs** — a pasted message, doc, or verbal description.

## File structure

```
one2ten/
  SKILL.md            # 5-phase workflow, gates, subagent orchestration rules
  references/
    planning.md       # plan format: bite-sized tasks, no placeholders, self-review
    subagents.md      # implementer/reviewer dispatch prompts + status contract
    n8n.md            # n8n MCP playbook
    make.md           # Make MCP playbook
    supabase.md       # Supabase MCP playbook
    code.md           # code repos / web apps / Vapi playbook
```

`SKILL.md` stays lean; references load only when relevant (per-platform
playbooks only for systems the ticket touches; planning/subagents files at
their phase).

## The five phases

### 1. Understand

- Fetch the ticket (Zoho MCP) or restate the chat brief back.
- Classify: **feature / task / bug**. Identify affected systems
  (n8n / Make / Supabase / code).
- **Bugs get evidence first:** pull failing executions (`n8n
  search_executions`, Make `executions_list`), logs (Supabase `get_logs`),
  reproduce the failure, and identify root cause *before* planning — the
  systematic-debugging principle: no fixes proposed until the cause is
  demonstrated, not guessed.
- Ask clarifying questions only when a requirement is genuinely ambiguous;
  otherwise proceed.

### 2. Plan (superpowers writing-plans method)

- Explore current state of every affected system first: list/fetch the
  workflows, scenarios, tables, functions the ticket touches.
- Write a plan document to `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md` in
  the working project, using the writing-plans rules:
  - Header: goal, architecture/approach, systems touched, global constraints.
  - **Bite-sized tasks:** each task is the smallest unit with its own
    verification cycle. Steps are single actions with exact content — exact
    node configs, SQL, module parameters, file paths, expected test output.
  - **No placeholders:** no "TBD", "add error handling", "similar to task N".
  - Each task declares **Files/Targets** (exact workflow names/IDs, table
    names, file paths), **Interfaces** (what it consumes from and produces
    for neighboring tasks), and a **Verify** step (the command/tool call and
    its expected result).
  - Risky or destructive actions flagged explicitly in the plan.
- Self-review the plan (coverage vs. ticket, placeholder scan, consistency
  of names/IDs across tasks). Fix inline.
- **GATE:** present the plan summary + path. Wait for approval. This is the
  only routine stop.

### 3. Build (subagent-driven, adapted for MCP work)

Orchestrator (main session) dispatches subagents; it does not implement.

- **Fresh implementer subagent per task.** Dispatch carries: one line of
  scene-setting, the task's text (as a brief file path, not pasted), the
  interfaces/decisions from earlier tasks, and the report contract.
  Implementer reports `DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED`.
- **Reviewer subagent after each task.** Verifies spec compliance and
  quality:
  - **Code work:** reviews the git diff (review-package pattern; record BASE
    commit before dispatch).
  - **MCP work (n8n/Make/Supabase):** there is no git diff — the reviewer
    *fetches live state* (workflow JSON, scenario blueprint, schema/
    migration list) via MCP and checks it against the task brief, and checks
    the implementer's test-execution evidence.
- **Fix loop:** Critical/Important findings → fix subagent → re-review.
  Minor findings recorded in the ledger for the final review to triage.
- **Progress ledger** at `.one2ten/progress.md`: one line per completed task
  (task, commits or workflow/scenario IDs + version, review result).
  Survives compaction; on resume, trust the ledger, never re-run completed
  tasks.
- **Model selection:** cheapest tier for transcription-grade tasks (plan
  contains the full content), standard for integration work, most capable
  for the final review. Specify explicitly on every dispatch.
- Tasks run sequentially by default; independent tasks touching disjoint
  systems (e.g., a Supabase migration and an unrelated Make scenario) may
  run in parallel.
- Safe-change rules bind all subagents: fetch current state before editing
  anything live; prefer additive changes; validate before publishing (n8n
  `validate_workflow`, Make blueprint validation); Supabase schema changes
  via `apply_migration`, never raw DDL through `execute_sql`; never
  deactivate/delete production assets without explicit human approval.

### 4. Test (verification-before-completion)

- After all tasks pass their per-task review, run end-to-end verification of
  the *ticket's* acceptance criteria — not just per-task checks:
  - n8n: `test_workflow` / real execution + `get_execution` inspection.
  - Make: `scenarios_run` + execution detail check.
  - Supabase: `execute_sql` assertions, `get_logs`, `get_advisors`.
  - Code: run the test suite and exercise the changed flow.
- Dispatch a **final reviewer subagent** (most capable model) for the whole
  change set: full branch diff for code, live-state audit for MCP work.
- Evidence rule: a claim of "works" requires the command/tool call and its
  observed output. Fix-and-retest loop until green.

### 5. Report

- Chat summary only (no automatic Zoho updates; Nat handles the tracker):
  - What was done, per system, with exact names/IDs of changed assets.
  - How it was verified (evidence: executions, test output).
  - Anything left, risks, or follow-ups.
  - For bugs: root cause in one sentence.

## Cross-cutting rules

- Stop for: destructive/irreversible actions, missing credentials (name
  exactly what's needed), genuine requirement ambiguity, or a BLOCKED status
  the orchestrator cannot resolve. Nothing else pauses the run after plan
  approval.
- Never report done without Test-phase evidence.
- Subagents get curated context, never the session history; artifacts move
  as files (briefs, reports, review packages), not pasted text.

## Out of scope (v1)

- Automatic Zoho status updates or comments.
- Vault note-writing to AllAboutNat.
- Multi-ticket batching; one ticket per run.
