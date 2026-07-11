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
- If they choose install: wait for the install and `codex login` to
  finish, re-probe, then proceed (mode ON on success).
- Record the outcome as the FIRST line of `.one2ten/progress.md`
  (create `.one2ten/` if it doesn't exist yet), picking ONE reason:
  `Codex mode: on` or `Codex mode: fallback (declined | auth failed |
  not installed)` — `declined` when the user chose the fallback at the
  ask. Resumed runs trust this line; never re-ask mid-run.

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
   under the Report's Follow-ups / risks. A PASS whose findings still
   include a Critical/Important item is a FAIL — the findings govern,
   not the verdict line.
3. FAIL with Critical/Important findings → fix them (plan gate: edit the
   plan; final: run the normal fix loop from `references/subagents.md`),
   then re-challenge. The round N+1 prompt appends a section
   `PRIOR FINDINGS AND FIXES:` listing each finding and what changed — so
   Codex verifies the fixes instead of rediscovering them.
4. Round 3 still FAIL → **stop condition**: stop and give the human the
   unresolved findings verbatim plus the round-file paths.

Never silently overrule a Critical/Important Codex finding. If you believe
a finding is wrong, that disagreement goes to the human — at the gate
(plan) or as a stop (final) — never self-adjudicated. Disputing a finding
ends the loop early: present the disagreement instead of fixing it; don't
burn the remaining rounds.

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
