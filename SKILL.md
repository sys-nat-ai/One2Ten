---
name: one2ten
description: Use when given a Zoho Projects task or issue, a pasted ticket or bug report, or asked to implement, do, or fix a task end to end across n8n workflows, Make scenarios, Supabase, code/apps, or Vapi voice agents via MCPs. Also use when resuming a ticket a previous session left unfinished (a .one2ten/ directory exists in the project).
---

# One2Ten — Ticket Workflow

**Announce at start:** "Using one2ten to run this ticket end-to-end."

Run one ticket per invocation through five phases: **Understand → Plan →
Build → Test → Report**. There is exactly ONE routine stop: plan approval.
After that gate, run to completion unless a stop condition (below) fires.

## Phase 0 — Intake

- **Resume check (always first):** look for `.one2ten/` in the working
  project. If `.one2ten/plans/` holds a plan for THIS ticket and/or
  `.one2ten/progress.md` has entries for it, this is a resumed run: read
  both, trust the ledger and live state over your own memory, and re-enter
  at the first incomplete point. An already-approved plan is NOT
  re-approved; tasks in the ledger are NEVER re-run — jump to Build at the
  first task missing from the ledger (or to Test if all tasks are ledgered).
- **Vault check:** walk up from the working project for an `.obsidian/`
  directory. Inside an Obsidian vault → read `references/vault.md`, then
  read the project's index note (`Projects/<project>.md` in the vault) —
  its assets, conventions, and open follow-ups are Understand/Plan input.
- **Codex check:** probe for the Codex CLI (`command -v codex`). Found →
  Codex mode ON: Codex independently challenges the plan (Phase 2) and
  the final change set (Phase 4) per `references/codex.md`. Not found (or
  auth fails at first use) → ask ONCE: install it (`npm install -g
  @openai/codex`, then `codex login`) or run this ticket with Claude
  self-review as fallback. Record the outcome as the first line of
  `.one2ten/progress.md` (`Codex mode: on` / `Codex mode: fallback
  (<reason>)`); never re-ask mid-run.
- **Zoho ticket:** if the input references a Zoho Projects task or issue,
  fetch it via the Zoho Projects MCP (`get_task_details`, `get_issue`; use
  portal/project lookups like `get_portals`, `get_projects_list` if IDs are
  missing). Quote the ticket title and requirements back.
- **Chat brief:** restate the request back in 2–4 sentences so
  misunderstandings surface now, not after Build.
- Classify the ticket: **feature** (new capability), **task** (change or
  chore), or **bug** (something broken).
- List the affected systems: n8n / Make / Supabase / code or Vapi. This
  drives which references you load.

## Reference load-table

Load only what this ticket needs, at the phase that needs it:

| When | Read |
|------|------|
| Entering Plan phase | `references/planning.md` |
| Entering Build phase | `references/subagents.md` |
| Ticket touches n8n | `references/n8n.md` |
| Ticket touches Make | `references/make.md` |
| Ticket touches Supabase | `references/supabase.md` |
| Ticket touches code / apps / Vapi | `references/code.md` |
| Project is inside an Obsidian vault (`.obsidian/` in an ancestor) | `references/vault.md` |
| Codex mode is on (or still undecided) | `references/codex.md` (at Intake, Plan, and Test) |

## Phase 1 — Understand

- Confirm the goal, the acceptance criteria (they become the plan's
  `## Acceptance Criteria` section — Test phase verifies exactly these),
  and the affected systems.
- **Bugs get evidence first.** Before any planning: pull the failing
  executions and logs (see the platform playbook), reproduce the failure,
  and demonstrate the root cause. No fix is proposed until the cause is
  demonstrated, not guessed. If you cannot reproduce it, say so and ask for
  the failing case — that is a valid stop.
- Ask clarifying questions ONLY for genuine ambiguity that changes what you
  would build. Otherwise proceed.

## Phase 2 — Plan

1. Read `references/planning.md`.
2. Explore the current state of every affected system first — list and
   fetch the actual workflows, scenarios, tables, functions, and files the
   ticket touches. Plans written blind get rejected by reality.
3. Write the plan to `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md` in the
   working project, following the planning reference exactly.
4. Self-review the plan (coverage, placeholders, name/ID consistency).
5. **Codex challenge (Codex mode):** run the plan-challenge loop per
   `references/codex.md` — fix Critical/Important findings and
   re-challenge until `VERDICT: PASS` (or Minor-only), max 3 rounds.
6. **GATE:** present a short plan summary, the plan file path, and the
   Codex verdict + round count (or the fallback note). Wait for approval.
   This is the only routine stop in the whole run.

## Phase 3 — Build

Read `references/subagents.md` and orchestrate:

- Fresh **implementer subagent** per task, dispatched with a task brief
  file — never the whole plan, never session history.
- **Reviewer subagent** after each task: spec compliance + quality. Code
  changes are reviewed by git diff from the recorded BASE; n8n/Make/
  Supabase changes are reviewed by fetching live state via MCP.
- Fix loop for Critical/Important findings; ledger every completed task in
  `.one2ten/progress.md`.
- The orchestrator (you) never implements directly — dispatch, review,
  decide.

## Phase 4 — Test

- Verify the **plan's `## Acceptance Criteria` end-to-end**, not just the
  per-task checks: run the workflow/scenario for real, query the data,
  exercise the app flow. Platform specifics are in each playbook.
- **Final whole-change review:** in Codex mode, run the final challenge
  loop per `references/codex.md` (branch diff for code; exported live
  state for MCP work), fixing and re-challenging until PASS, max 3
  rounds. Fallback (Codex unavailable, declined, or errored): dispatch a
  **final reviewer subagent** on the most capable model to audit the
  whole change set (full branch diff for code; live-state audit for MCP
  work).
- **Evidence rule:** every "it works" claim must carry the tool call or
  command AND its observed output. No evidence, no claim.
- Loop fix-and-retest until green.

## Phase 5 — Report

Chat summary only — no automatic Zoho updates; Nat updates the tracker.
In vault mode, also update the project's index note per
`references/vault.md`, and end the Report with a **Vault note updated**
section quoting exactly the lines added or changed.
Use this shape:

```
## Done: <ticket title>

**What changed** (per system, exact asset names/IDs)
- n8n: <workflow name (ID)> — <what changed>
- Supabase: <migration name / table> — <what changed>
- ...

**How it was verified** (evidence)
- <tool call / command> → <observed result>

**Challenged by:** <Codex — plan: N round(s), final: N round(s) (PASS) |
Claude self-review (<reason>)>

**Root cause** (bugs only, one sentence)

**Follow-ups / risks**
- <anything left, or "None">
```

## Cross-cutting rules

**Safe-change rules** (bind you and every subagent):
- Fetch current state before editing anything live.
- Save a rollback copy of any live asset (workflow JSON, blueprint,
  function source, assistant config) to `.one2ten/briefs/` before editing.
- Prefer additive changes over rewrites.
- Validate before publishing (n8n `validate_workflow`, Make
  `validate_blueprint_schema`).
- Supabase schema changes via `apply_migration` — never raw DDL through
  `execute_sql`.
- Never deactivate/delete production assets without explicit human
  approval.

**Stop conditions** (the only pauses after plan approval):
- A destructive or irreversible action is required.
- Missing credentials or access — name exactly what is needed, then stop.
- Genuine requirement ambiguity that changes what gets built.
- A BLOCKED subagent status the orchestrator cannot resolve.
- A Codex challenge still returns FAIL after 3 rounds — surface the
  unresolved findings verbatim with the round-file paths.

**Always:**
- Never report done without Phase 4 evidence.
- Subagents get curated context handed over as files (briefs, reports,
  diffs) — never pasted session history.
