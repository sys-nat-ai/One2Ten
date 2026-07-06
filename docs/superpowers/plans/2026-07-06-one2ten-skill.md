# One2Ten Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `/one2ten` personal skill — Nat's end-to-end ticket workflow (Understand → Plan → Build → Test → Report) — as SKILL.md + 6 reference files, versioned in this repo and installed to `C:\Users\Nat Pisco\.claude\skills\one2ten`.

**Architecture:** Repo root holds `SKILL.md` and `references/`. SKILL.md is the lean orchestrator document (phases, gate, rules, and a load-table saying which reference to read when). Each reference file is self-contained guidance for one concern. Install = copy SKILL.md + references/ to the skills directory.

**Tech Stack:** Markdown skill files (Claude Code skill format: YAML frontmatter with `name` and `description`), PowerShell for install/verify.

## Global Constraints

- Skill name is exactly `one2ten`; installed path `C:\Users\Nat Pisco\.claude\skills\one2ten\`.
- Flow is Understand → Plan → Build → Test → Report with exactly ONE routine human gate: plan approval.
- Plans written by the skill go to `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md` in the target project; progress ledger at `.one2ten/progress.md`.
- Implementer status vocabulary is exactly: `DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, `BLOCKED`.
- MCP review rule: for n8n/Make/Supabase work reviewers fetch live state via MCP (no git diff exists); for code work reviewers use git diff from a recorded BASE commit.
- Safe-change rules (verbatim in SKILL.md): fetch current state before editing anything live; prefer additive changes; validate before publishing; Supabase schema via `apply_migration` never raw DDL through `execute_sql`; never deactivate/delete production assets without explicit human approval.
- Report step is chat-summary only — no automatic Zoho updates (v1).
- No placeholders in any file: no TBD/TODO, every rule stated concretely with exact MCP tool names.

## File Structure

```
One2Ten/                          # this repo (source of truth)
  SKILL.md                        # orchestrator: phases, gate, rules, reference load-table
  references/
    planning.md                   # plan document format (writing-plans method, MCP-adapted)
    subagents.md                  # implementer/reviewer/fixer dispatch contracts, ledger, models
    n8n.md                        # n8n MCP playbook
    make.md                       # Make MCP playbook
    supabase.md                   # Supabase MCP playbook
    code.md                       # code repos / apps / Vapi playbook
  docs/superpowers/...            # spec + this plan (not installed)
```

---

### Task 1: SKILL.md

**Files:**
- Create: `SKILL.md` (repo root)

**Interfaces:**
- Produces: the reference load-table used by Tasks 2–7 — file names must match exactly: `references/planning.md`, `references/subagents.md`, `references/n8n.md`, `references/make.md`, `references/supabase.md`, `references/code.md`.

- [ ] **Step 1: Write SKILL.md** with frontmatter exactly:

```yaml
---
name: one2ten
description: End-to-end AI-engineer ticket workflow - implement, do, or fix a task ticket across n8n, Make, Supabase, and code/Vapi via MCPs. Use when given a Zoho Projects task/issue, a pasted ticket or bug report, or asked to implement/fix/do a task end to end. Flow - Understand, Plan (one approval gate), Build (subagent-driven), Test (evidence required), Report (chat summary).
---
```

Body sections, in order, each concrete:
1. **Announce**: "Using one2ten to run this ticket end-to-end."
2. **Phase 0 — Intake**: if input references Zoho, fetch via Zoho Projects MCP (`get_task_details` / `get_issue`, portal/project lookups as needed); if chat brief, restate it back in 2–4 sentences. Classify feature/task/bug. List affected systems (n8n / Make / Supabase / code-Vapi).
3. **Reference load-table** (markdown table): Plan phase → read `references/planning.md`; Build phase → read `references/subagents.md`; any phase touching a platform → read that platform's file (`references/n8n.md`, `references/make.md`, `references/supabase.md`, `references/code.md`). Load only what the ticket touches.
4. **Phase 1 — Understand**: bugs get evidence first — pull failing executions/logs, reproduce, demonstrate root cause BEFORE planning (systematic-debugging rule: no fix proposed until cause is demonstrated, not guessed). Clarifying questions only for genuine ambiguity.
5. **Phase 2 — Plan**: explore current state of every affected system first; write plan per `references/planning.md` to `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md`; self-review; **GATE — present plan summary + path, wait for approval. Only routine stop in the whole run.**
6. **Phase 3 — Build**: orchestrate per `references/subagents.md` — fresh implementer subagent per task, reviewer subagent per task, fix loop, ledger. Orchestrator never implements directly.
7. **Phase 4 — Test**: verify the TICKET's acceptance criteria end-to-end (not just per-task checks); final whole-change reviewer subagent on most capable model; evidence rule — every "works" claim carries the tool call/command AND its observed output.
8. **Phase 5 — Report**: chat summary template — what changed (exact asset names/IDs per system), how verified (evidence), root cause (bugs, one sentence), follow-ups/risks. No automatic Zoho updates.
9. **Cross-cutting rules**: the safe-change rules from Global Constraints verbatim; stop conditions (destructive action, missing credentials — name exactly what's needed, genuine ambiguity, unresolvable BLOCKED); never report done without Test-phase evidence; subagents get curated context via files, never session history.

- [ ] **Step 2: Verify** — file exists, frontmatter has exactly `name: one2ten` and one-line description, all six reference paths in the load-table match the File Structure list.

- [ ] **Step 3: Commit**

```powershell
git add SKILL.md; git commit -m "feat: one2ten SKILL.md orchestrator"
```

### Task 2: references/planning.md

**Files:**
- Create: `references/planning.md`

**Interfaces:**
- Consumes: plan path convention `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md` (from SKILL.md).
- Produces: plan document format that `references/subagents.md` task briefs are extracted from.

- [ ] **Step 1: Write planning.md** containing:
- Plan header format: Goal (one sentence), Approach (2–3 sentences), Systems touched, Global Constraints section (exact values from ticket — URLs, IDs, names, formats).
- Task structure block (verbatim template): `**Targets:**` (exact workflow names+IDs / scenario IDs / table names / file paths), `**Interfaces:**` (Consumes/Produces with exact names, signatures, field shapes), checkbox steps, final `**Verify:**` step naming the exact tool call or command and its expected result.
- Bite-sized rule: each step one action; each task the smallest unit with its own verification cycle.
- No-placeholders list (verbatim failures): "TBD", "TODO", "add error handling", "handle edge cases", "similar to Task N", steps that describe without showing content.
- MCP adaptation: for n8n tasks steps contain actual node configuration values; for Supabase, the actual SQL; for Make, actual module parameters; for code, actual code.
- Risky/destructive actions flagged with a `⚠️ DESTRUCTIVE` marker in the task that performs them.
- Self-review checklist: ticket coverage, placeholder scan, name/ID consistency across tasks.

- [ ] **Step 2: Verify** — every rule above appears; no placeholder patterns present in the file itself.

- [ ] **Step 3: Commit** — `git add references/planning.md; git commit -m "feat: planning reference"`

### Task 3: references/subagents.md

**Files:**
- Create: `references/subagents.md`

**Interfaces:**
- Consumes: plan/task format from `references/planning.md`; ledger path `.one2ten/progress.md`.
- Produces: dispatch contracts used during Build; status vocabulary `DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED`.

- [ ] **Step 1: Write subagents.md** containing:
- Orchestrator role: dispatch, review, decide; never implement directly; narrate one short line between dispatches.
- Implementer dispatch template (verbatim skeleton): 1 line scene-setting; task brief as file path (extract the task's text from the plan to `.one2ten/briefs/task-N-brief.md`, tell the subagent "read this first — it is your requirements"); interfaces/decisions from earlier tasks; report contract (write full report to `.one2ten/briefs/task-N-report.md`, return only status + one-line summary + concerns); status vocabulary with meanings.
- Status handling: DONE → review; DONE_WITH_CONCERNS → read concerns first; NEEDS_CONTEXT → supply and re-dispatch; BLOCKED → more context / stronger model / split task / escalate to human — never silent retry.
- Reviewer dispatch template: gets brief path + report path + the change evidence. **Code work:** record BASE commit before implementer dispatch; reviewer gets `git diff BASE..HEAD` written to a file (never `HEAD~1`). **MCP work:** reviewer fetches live state itself — n8n `get_workflow_details`, Make `scenarios_get` blueprint, Supabase `list_migrations`/`list_tables` — and checks it against the brief plus the implementer's execution evidence. Two verdicts required: spec compliance AND quality.
- Fix loop: Critical/Important → fix subagent → re-review; Minor → ledger, triaged at final review. Never proceed with open Critical/Important. Never tell a reviewer what not to flag.
- Ledger `.one2ten/progress.md`: append one line per completed task — `Task N: complete (commits X..Y | workflow/scenario ID + change, review clean)`; on resume trust ledger + git log/MCP state over memory; never re-run a ledgered task.
- Model selection: cheapest tier when the plan contains the full content (transcription); standard for integration; most capable for final review; specify model on every dispatch.
- Parallelism: sequential by default; parallel only for independent tasks on disjoint systems.

- [ ] **Step 2: Verify** — all four statuses defined; both reviewer modes (git diff / live MCP state) present; ledger format shown.

- [ ] **Step 3: Commit** — `git add references/subagents.md; git commit -m "feat: subagent orchestration reference"`

### Task 4: references/n8n.md

**Files:**
- Create: `references/n8n.md`

- [ ] **Step 1: Write n8n.md** playbook:
- Explore first: `search_workflows`, `get_workflow_details` (full JSON before ANY edit), `search_executions` + `get_execution` for debugging evidence.
- Before building: `get_sdk_reference` (mandatory before writing SDK code), `get_workflow_best_practices` per technique, `search_nodes` → `get_node_types` with discriminators (never guess parameters), `explore_node_resources` for `@searchListMethod`/`@loadOptionsMethod` values (never invent IDs), `list_credentials` for credential IDs.
- Build: `create_workflow_from_code` / `update_workflow`; validate with `validate_node_config` and `validate_workflow` before publish.
- Test: `prepare_test_pin_data` + `test_workflow`, inspect with `get_execution`; only then `publish_workflow`.
- Safety: never `update_workflow` on a live workflow without fetching current details first; keep a copy of prior JSON in `.one2ten/briefs/` as rollback; `unpublish_workflow`/`archive_workflow` are approval-gated.
- Expression/code gotchas: defer to the n8n-expression-syntax and n8n-code-javascript skills when writing expressions or Code nodes.

- [ ] **Step 2: Verify** — every tool name above appears and matches the MCP tool list.
- [ ] **Step 3: Commit** — `git add references/n8n.md; git commit -m "feat: n8n playbook"`

### Task 5: references/make.md

**Files:**
- Create: `references/make.md`

- [ ] **Step 1: Write make.md** playbook:
- Explore: `scenarios_list`, `scenarios_get` (blueprint before any edit), `executions_list` + `executions_get-detail` for evidence, `connections_list` to confirm credentials exist.
- Build: `apps_recommend`/`apps_list` → `app-modules_list` → `app-module_get` + `app_documentation_get` for exact module parameters; assemble blueprint; `validate_blueprint_schema` + `validate_module_configuration` before `scenarios_create`/`scenarios_update`; data stores via `data-stores_*` and `data-structures_*`; webhooks via `hooks_*` with `validate_hook_configuration`.
- Test: `scenarios_run`, then `executions_get-detail` on the run; check every module's output, not just scenario status.
- Safety: fetch and save the current blueprint before updating a live scenario (rollback copy in `.one2ten/briefs/`); `scenarios_deactivate`/`scenarios_delete` are approval-gated; scheduling changes validated with `validate_scheduling_schema`.

- [ ] **Step 2: Verify** — tool names match the Make MCP tool list.
- [ ] **Step 3: Commit** — `git add references/make.md; git commit -m "feat: Make playbook"`

### Task 6: references/supabase.md

**Files:**
- Create: `references/supabase.md`

- [ ] **Step 1: Write supabase.md** playbook:
- Explore: `list_projects`/`get_project`, `list_tables`, `list_migrations`, `list_extensions`, `list_edge_functions`/`get_edge_function` — always before schema changes.
- Debug first: `get_logs` (by service), `get_advisors` — before changing anything on a bug ticket.
- Build: schema/DDL ONLY via `apply_migration` (named migration); data fixes and verification queries via `execute_sql`; edge functions via `deploy_edge_function`; types via `generate_typescript_types` when the project consumes them.
- Risky changes: use `create_branch` → apply → verify → `merge_branch`; `reset_branch`/`delete_branch` for abandoning.
- Test: `execute_sql` assertions with expected rows stated; `get_logs` after deploy; `get_advisors` after schema changes (fix new security/performance advisories or report them).
- Safety: DROP/TRUNCATE/DELETE-without-WHERE and `pause_project`/`restore_project` are approval-gated; never run destructive SQL through `execute_sql` casually.

- [ ] **Step 2: Verify** — tool names match the Supabase MCP tool list.
- [ ] **Step 3: Commit** — `git add references/supabase.md; git commit -m "feat: Supabase playbook"`

### Task 7: references/code.md

**Files:**
- Create: `references/code.md`

- [ ] **Step 1: Write code.md** playbook:
- Features/tasks: TDD loop per task (failing test → minimal code → pass → commit); follow existing repo patterns; frequent small commits.
- Bugs: systematic-debugging — reproduce, read the actual error, form hypothesis, prove root cause with evidence, THEN fix; the fix includes a regression test.
- Verification: run the project's real test command; exercise the changed flow end-to-end (run the app), not just unit tests.
- Vapi voice agents: `list_assistants`/`get_assistant` before editing; `update_assistant` on a live agent only after saving current config as rollback; tools via `create_tool`/`update_tool`/`get_tool`; verify with `create_call`/`get_call` where a test call is feasible, otherwise report config-level verification honestly.
- Git hygiene: work on a branch for multi-commit changes; never force-push or reset --hard without approval.

- [ ] **Step 2: Verify** — Vapi tool names match the vapi-mcp tool list.
- [ ] **Step 3: Commit** — `git add references/code.md; git commit -m "feat: code and Vapi playbook"`

### Task 8: Install and verify

**Files:**
- Create: `C:\Users\Nat Pisco\.claude\skills\one2ten\SKILL.md` + `references\*.md` (copies)

- [ ] **Step 1: Copy skill to personal skills directory**

```powershell
New-Item -ItemType Directory -Force "C:\Users\Nat Pisco\.claude\skills\one2ten\references"
Copy-Item "SKILL.md" "C:\Users\Nat Pisco\.claude\skills\one2ten\SKILL.md" -Force
Copy-Item "references\*.md" "C:\Users\Nat Pisco\.claude\skills\one2ten\references\" -Force
```

- [ ] **Step 2: Verify install** — `Get-ChildItem -Recurse "C:\Users\Nat Pisco\.claude\skills\one2ten"` shows SKILL.md + 6 reference files; SKILL.md first line is `---` and contains `name: one2ten`. Note: the skill registers in NEW sessions (or after reload); confirm by frontmatter validity now.

- [ ] **Step 3: Commit plan-completion note** — append install date to README or commit message only:

```powershell
git add -A; git commit -m "chore: install one2ten skill to personal skills directory"
```

## Self-Review

- Spec coverage: phases (T1), plan format (T2), subagent build (T3), platform playbooks (T4–7), install (T8) — all spec sections mapped. Report template lives in T1 §8. ✔
- Placeholder scan: none; every task names exact tools/paths. ✔
- Consistency: reference file names, ledger/brief paths (`.one2ten/...`), and status vocabulary identical across tasks. ✔
