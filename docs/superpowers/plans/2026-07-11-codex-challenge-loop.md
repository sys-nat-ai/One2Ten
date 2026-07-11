# Codex Challenge Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** OpenAI Codex becomes One2Ten's independent challenger at the plan
gate and the final review, via a bounded challenge loop.

**Architecture:** New `references/codex.md` playbook (setup check,
invocation, prompt templates, verdict contract, loop rules, fallback) +
surgical hooks in SKILL.md (Phase 0 check, load-table row, Phase 2 step,
Phase 4 replacement, stop condition, Report line). Direct `codex exec`
calls — no gstack, no MCP. Spec:
`docs/superpowers/specs/2026-07-11-codex-challenge-loop-design.md`.

**Tech Stack:** Markdown only (skill documentation); the playbook's
runtime commands use the Codex CLI (`codex exec`) and bash.

## Global Constraints

- Checkpoint names are exactly `plan` and `final`; round artifacts go to
  `.one2ten/briefs/codex-<checkpoint>-round-<N>-prompt.md` (prompt) and
  `.one2ten/briefs/codex-<checkpoint>-round-<N>.md` (output).
- Every Codex run: `-s read-only` sandbox, 10-minute timeout, filesystem
  boundary preamble, verdict contract ending `VERDICT: PASS|FAIL`.
- Max 3 rounds per checkpoint; round 3 FAIL is a stop condition.
- The setup ask happens ONCE, at intake, never mid-run; the outcome is
  recorded as the first line of `.one2ten/progress.md`.
- Codex failures never block a ticket — Claude review is always the
  fallback, and the Report's **Challenged by** line always states which
  path ran.
- Per-task Build reviewers stay Claude subagents (unchanged).

---

### Task 1: Create `references/codex.md`

**Files:**
- Create: `references/codex.md`

**Interfaces:**
- Produces: the playbook that SKILL.md's hooks (Task 2) point at, including
  the `Codex mode:` progress-ledger line format and the **Challenged by**
  Report line formats Task 2 references.

- [ ] **Step 1: Write the file with exactly this content**

`````markdown
# Codex Playbook — cross-model challenge loop

Claude reviewing Claude leaves shared blind spots. When the Codex CLI is
available, OpenAI Codex is the independent challenger at two checkpoints:
the **plan gate** (Phase 2) and the **final review** (Phase 4). Each
checkpoint is a bounded loop: challenge → fix Critical/Important findings
→ re-challenge, until PASS or 3 rounds are exhausted. Per-task Build
reviewers stay Claude subagents — the final Codex review backstops them.

## Setup check (Phase 0)

- Probe: `command -v codex` (bash).
- Found → Codex mode ON. No questions asked.
- Not found → ask ONCE, at intake:
  > Codex CLI powers One2Ten's independent bias check (it challenges the
  > plan and the final change set). Install it now —
  > `npm install -g @openai/codex` then `codex login` — or run this
  > ticket with Claude self-review as the fallback?
- Found but auth fails at first use → same ask, quoting the auth error.
- Record the outcome as the FIRST line of `.one2ten/progress.md`:
  `Codex mode: on` or `Codex mode: fallback (not installed | declined |
  auth failed)`. Resumed runs trust this line; never re-ask mid-run.

## Invocation pattern (both checkpoints)

1. Write the full prompt to
   `.one2ten/briefs/codex-<checkpoint>-round-<N>-prompt.md`
   (`<checkpoint>` ∈ `plan`, `final`). Prompts embed whole documents — a
   file avoids shell-quoting damage.
2. Run with a 10-minute timeout:

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
codex exec "$(cat .one2ten/briefs/codex-<checkpoint>-round-<N>-prompt.md)" \
  -C "$_ROOT" -s read-only \
  > .one2ten/briefs/codex-<checkpoint>-round-<N>.md
```

3. Read the saved output file — it is the review of record.

**Errors:** non-zero exit, timeout, or empty output → retry once. Second
failure → Claude fallback for THAT checkpoint; the Report line says so.
Codex problems never block the ticket.

## Prompt contract (every prompt)

Start with the filesystem boundary:

```
IMPORTANT: Do NOT read or execute any files under ~/.claude/, .claude/,
~/.agents/, or any skills/ directory. Work only with the repository files
and the artifact paths named below.
```

End with the verdict contract:

```
End your response with your findings, each rated Critical / Important /
Minor with a one-line reason and the exact location (file / task / node),
followed by a final line that is exactly `VERDICT: PASS` or
`VERDICT: FAIL`. PASS means: no Critical or Important findings.
```

Output missing the verdict line → re-run once; still missing → treat as
FAIL and use the raw output as the findings.

**Embed, don't reference.** Codex is sandboxed to the repo root and has NO
MCP access:

- Plan gate → embed the full plan file content in the prompt.
- Final review, code work → write `git diff <BASE>..HEAD` (BASE = the
  first task's recorded BASE commit) plus `git log --oneline <BASE>..HEAD`
  to `.one2ten/briefs/final-diff.txt` and name that path in the prompt.
- Final review, MCP work → export live state to `.one2ten/briefs/`
  (n8n workflow JSON via `get_workflow_details`, Make blueprint via
  `scenarios_get`, Supabase `list_migrations`/`list_tables` output,
  function source, Vapi assistant config) and name those paths, one line
  each on what it is. Codex reviews real exported state, never your
  summary of it.

## Plan-gate prompt (after self-review, before the approval gate)

```
<filesystem boundary>

You are a brutally honest technical reviewer for an automation ticket.
Attack this implementation plan: logical gaps and unstated assumptions,
missing error handling, placeholder-grade steps (anything that describes
without showing the real config/SQL/code), overcomplexity, feasibility
risks, wrong sequencing or missing dependencies between tasks, and
platform-specific risks (malformed n8n/Make node parameters, unsafe or
irreversible Supabase migrations, destructive steps not marked
⚠️ DESTRUCTIVE). Do not compliment. Problems only.

THE TICKET:
<ticket summary + acceptance criteria>

THE PLAN:
<full plan file content, verbatim>

<verdict contract>
```

## Final-review prompt (Phase 4)

```
<filesystem boundary>

You are an adversarial reviewer. Your job is to find how this change set
fails in production. Think like an attacker and a chaos engineer: edge
cases, race conditions, security holes, silent data corruption, resource
leaks, failure modes. Then check every acceptance criterion below against
the artifacts: covered or not, with evidence.

THE TICKET:
<ticket summary>

ACCEPTANCE CRITERIA:
<numbered list from the plan's ## Acceptance Criteria>

THE CHANGE SET — read these files:
<.one2ten/briefs/final-diff.txt and/or exported live-state paths, one per
line, each with one line on what it is>

<verdict contract>
```

## The loop (each checkpoint, max 3 rounds)

1. Challenge; save the round file.
2. `VERDICT: PASS`, or FAIL with only Minor findings → the checkpoint
   clears. Minors: plan gate → noted in the gate summary; final → listed
   under the Report's Follow-ups / risks.
3. FAIL with Critical/Important findings → fix them (plan gate: edit the
   plan; final: run the normal fix loop from `references/subagents.md`),
   then re-challenge. The round N+1 prompt appends a section
   `PRIOR FINDINGS AND FIXES:` listing each finding and what changed — so
   Codex verifies the fixes instead of rediscovering them.
4. Round 3 still FAIL → **stop condition**: stop and give the human the
   unresolved findings verbatim plus the round-file paths.

Never silently overrule a Critical/Important Codex finding. If you believe
a finding is wrong, that disagreement goes to the human — at the gate
(plan) or as a stop (final) — never self-adjudicated.

## Fallback (Codex mode off, or Codex errored)

- Plan gate: the planning reference's self-review only.
- Final review: the Claude final-reviewer subagent on the most capable
  model (per `references/subagents.md`).
- The Report must say so — a skipped bias check is visible, never silent.

## Report line (Phase 5)

Add after **How it was verified**:

- Codex mode:
  `**Challenged by:** Codex — plan: <N> round(s), final: <N> round(s) (PASS)`
- Fallback:
  `**Challenged by:** Claude self-review (Codex not installed | declined | auth failed)`
- Error path:
  `**Challenged by:** Claude self-review (Codex errored — fallback at <checkpoint>)`
`````

- [ ] **Step 2: Commit**

```bash
git add references/codex.md
git commit -m "feat: codex playbook - cross-model challenge loop"
```

### Task 2: SKILL.md hooks

**Files:**
- Modify: `SKILL.md` (Phase 0, load-table, Phase 2, Phase 4, stop
  conditions, Phase 5 report shape)

**Interfaces:**
- Consumes: `references/codex.md` (Task 1) — its `Codex mode:` ledger line
  and **Challenged by** formats.

- [ ] **Step 1: Add Phase 0 Codex check** — insert a bullet after the
  Vault check bullet:

```markdown
- **Codex check:** probe for the Codex CLI (`command -v codex`). Found →
  Codex mode ON: Codex independently challenges the plan (Phase 2) and
  the final change set (Phase 4) per `references/codex.md`. Not found (or
  auth fails at first use) → ask ONCE: install it (`npm install -g
  @openai/codex`, then `codex login`) or run this ticket with Claude
  self-review as fallback. Record the outcome as the first line of
  `.one2ten/progress.md` (`Codex mode: on` / `Codex mode: fallback
  (<reason>)`); never re-ask mid-run.
```

- [ ] **Step 2: Add load-table row** — after the Obsidian vault row:

```markdown
| Codex mode is on (or still undecided) | `references/codex.md` (at Plan and Test) |
```

- [ ] **Step 3: Insert the Phase 2 challenge step** — replace step 5
  (`**GATE:** present a short plan summary...`) with:

```markdown
5. **Codex challenge (Codex mode):** run the plan-challenge loop per
   `references/codex.md` — fix Critical/Important findings and
   re-challenge until `VERDICT: PASS` (or Minor-only), max 3 rounds.
6. **GATE:** present a short plan summary, the plan file path, and the
   Codex verdict + round count (or the fallback note). Wait for approval.
   This is the only routine stop in the whole run.
```

- [ ] **Step 4: Replace the Phase 4 final-reviewer bullet** — replace the
  bullet beginning `Dispatch a **final reviewer subagent**...` with:

```markdown
- **Final whole-change review:** in Codex mode, run the final challenge
  loop per `references/codex.md` (branch diff for code; exported live
  state for MCP work), fixing and re-challenging until PASS, max 3
  rounds. Fallback (Codex unavailable, declined, or errored): dispatch a
  **final reviewer subagent** on the most capable model to audit the
  whole change set (full branch diff for code; live-state audit for MCP
  work).
```

- [ ] **Step 5: Add a stop condition** — append to the stop-conditions
  list:

```markdown
- A Codex challenge still returns FAIL after 3 rounds — surface the
  unresolved findings verbatim with the round-file paths.
```

- [ ] **Step 6: Add the Challenged-by line to the report shape** — in the
  Phase 5 report template, insert between the `**How it was verified**`
  block and the `**Root cause**` line:

```markdown
**Challenged by:** <Codex — plan: N round(s), final: N round(s) (PASS) |
Claude self-review (<reason>)>
```

- [ ] **Step 7: Commit**

```bash
git add SKILL.md
git commit -m "feat: codex hooks - intake check, plan-gate + final challenge loops"
```

### Task 3: README updates

**Files:**
- Modify: `README.md` (Why bullets, install prompt, repo layout,
  requirements)

- [ ] **Step 1: Add a Why bullet** after the second-brain bullet:

```markdown
- ⚔️ **Cross-model challenged** — with the OpenAI Codex CLI installed,
  Codex independently attacks the plan before approval and the final
  change set before "done" — no model grading its own homework
```

- [ ] **Step 2: Update the agent-install prompt** — in Option A step 3,
  extend the references list to include `codex.md`:
  `(planning.md, subagents.md, n8n.md, make.md, supabase.md, code.md,
  vault.md, codex.md)`; in step 4, change `7 files under references/` to
  `8 files under references/`.

- [ ] **Step 3: Add `codex.md` to the repo-layout block** after `vault.md`:

```text
  codex.md                # ⚔️ Codex challenge-loop playbook (cross-model review)
```

- [ ] **Step 4: Add an optional requirement** — append to the
  Requirements list:

```markdown
- *(Optional)* the **OpenAI Codex CLI** (`npm install -g @openai/codex` +
  `codex login`) — enables the cross-model challenge loop; without it the
  skill falls back to Claude self-review and says so in the report
```

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: README - codex challenge loop"
```

### Task 4: Probe + install sync

- [ ] **Step 1: Consistency read-through** — read the final SKILL.md and
  `references/codex.md` together and verify: the ledger line format
  (`Codex mode: ...`) matches in both files; checkpoint names are `plan`
  and `final` everywhere; the Report shape's Challenged-by line matches
  the playbook's formats; Phase 2 step numbering is sequential (1–6).
  Fix any drift and amend the relevant commit message style (`fix:`).

- [ ] **Step 2: Probe the no-codex path** — dispatch a subagent probe:
  give it SKILL.md + references/codex.md and a scenario ("new ticket,
  `command -v codex` returns nothing") and ask what the skill does next.
  Expected: a single intake ask offering install vs fallback, outcome
  recorded as the first `progress.md` line, no re-ask later, fallback
  wording in the Report. If it misreads, tighten wording and re-probe.

- [ ] **Step 3: Probe the codex path** — same probe with "codex found;
  plan self-review done". Expected: prompt written to
  `.one2ten/briefs/codex-plan-round-1-prompt.md`, `codex exec … -s
  read-only` run, verdict parsed, loop/stop behavior per playbook, gate
  summary includes verdict + rounds.

- [ ] **Step 4: Sync the installed copy**

```powershell
Copy-Item "SKILL.md" "$env:USERPROFILE\.claude\skills\one2ten\SKILL.md" -Force
Copy-Item "references\*.md" "$env:USERPROFILE\.claude\skills\one2ten\references\" -Force
```

Expected: `references/` in the install dir now contains 8 files including
`codex.md`.
