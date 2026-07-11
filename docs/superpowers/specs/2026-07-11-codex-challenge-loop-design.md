# One2Ten — Codex Challenge Loop — Design

**Date:** 2026-07-11
**Owner:** Nat (AI Engineer)
**Status:** Approved (design approved in chat)

## What this adds

One2Ten currently has Claude reviewing Claude at every quality gate — the
same model family generates the plan, self-reviews it, implements it, and
audits the result, so blind spots survive every checkpoint. This change
introduces a **cross-model challenge loop**: OpenAI Codex (a genuinely
independent model, invoked via the Codex CLI) becomes the adversarial
challenger at the two highest-leverage checkpoints:

1. **Plan gate** — before Nat is asked to approve the plan, Codex
   challenges it. Bad plans get caught while fixing them is cheap.
2. **Final review** — in the Test phase, Codex adversarially reviews the
   whole change set instead of a Claude subagent.

Each checkpoint runs a bounded loop: challenge → Claude fixes
Critical/Important findings → re-challenge, until Codex passes or 3 rounds
are exhausted. Per-task reviewers in the Build phase stay as Claude
subagents (fast, cheap, and the final Codex review backstops them).

Explicitly NOT in scope (per Nat's choices):
- Codex per-task reviewers (too slow per ticket; option rejected).
- Invoking the gstack `/codex` skill (interactive preamble conflicts with
  One2Ten's single-approval-gate design; adds a gstack dependency; its
  prompts assume a git diff, which MCP tickets don't have). One2Ten calls
  `codex exec` directly with its own prompts.
- Hard-requiring Codex. It is asked for once; Claude self-review is the
  explicit, reported fallback.

## Architecture

- New reference: `references/codex.md` — setup check, invocation pattern,
  challenge prompt templates, verdict format, loop rules, fallback rules.
  Loads at Plan and Test phases when Codex mode is on (or being decided).
- `SKILL.md` hooks: Codex check in Phase 0, load-table row, plan-challenge
  step in Phase 2 (before the gate), final-review replacement in Phase 4,
  a new stop condition, and a **Challenged by** line in the Report.
- No other files change. `references/planning.md` self-review still runs
  first — it is cheap and removes placeholders before Codex tokens are
  spent. `references/subagents.md` per-task flow is untouched.

## Behavior

### Setup check (Phase 0 — Intake)

Probe for the `codex` binary (`command -v codex`). Three outcomes:

- **Found** → Codex mode ON for the run. No questions asked.
- **Not found** → ask Nat ONCE, at intake (not mid-run):
  > "Codex CLI powers One2Ten's independent bias check (it challenges the
  > plan and the final change set). Install it now —
  > `npm install -g @openai/codex` then `codex login` — or run this ticket
  > with Claude self-review as the fallback?"
  The answer is recorded for the entire run and never re-asked.
- **Found but auth fails at first use** → treat like "not found": surface
  the auth error, offer `codex login` or fallback, once.

This is the only new interaction; after the plan approval gate the run
remains autonomous.

### Invocation pattern (both checkpoints)

- Direct CLI call — no gstack, no MCP:
  `codex exec "<prompt>" -C <repo root> -s read-only` with a 10-minute
  timeout. Output is saved verbatim to
  `.one2ten/briefs/codex-<checkpoint>-round-<N>.md`
  (`<checkpoint>` ∈ `plan`, `final`).
- **Filesystem boundary preamble** starts every prompt: do not read or
  execute files under `~/.claude/`, `.claude/`, `~/.agents/`, or skill
  directories; stay on repository/ticket artifacts only.
- **Verdict contract** ends every prompt: the response MUST end with a
  line `VERDICT: PASS` or `VERDICT: FAIL`, preceded by findings each
  rated Critical / Important / Minor with a one-line reason. A response
  missing the verdict line is re-run once; if still missing, treat as
  FAIL with the raw output as findings.
- **Embed, don't reference.** Codex is sandboxed to the repo and has no
  MCP access:
  - Plan gate → the full plan file content is embedded in the prompt.
  - Final review, code work → write `git diff BASE..HEAD` to
    `.one2ten/briefs/final-diff.txt` and name that path in the prompt
    (it is inside the repo, so Codex can read it).
  - Final review, MCP work → export live state (workflow JSON, Make
    blueprint, migrations list, function source — the same artifacts
    already saved as rollback copies) into `.one2ten/briefs/` and name
    those paths. Codex reviews real exported state, never Claude's
    summary of it.

### Challenge prompts

- **Plan gate:** ticket summary + acceptance criteria + full plan,
  attacked for: logical gaps and unstated assumptions, missing error
  handling, placeholder-grade steps, overcomplexity, feasibility risks,
  sequencing/dependency errors, and platform-specific risks (wrong node
  config shapes, unsafe migrations, destructive steps not marked).
- **Final review:** adversarial framing — "find how this fails in
  production" — over the diff and/or exported live state, explicitly
  checked against the plan's `## Acceptance Criteria` (each criterion:
  covered or not, with evidence from the artifacts).

### The loop (each checkpoint, max 3 rounds)

1. Run the challenge. Save output to the round file.
2. `VERDICT: PASS`, or FAIL with only Minor findings → checkpoint clears.
   Minors are recorded (plan gate: noted in the gate summary; final:
   listed under Follow-ups / risks in the Report).
3. FAIL with Critical/Important findings → Claude fixes them (plan gate:
   edits the plan; final: dispatches fix subagents per the existing fix
   loop), then re-challenges. The re-challenge prompt includes the prior
   findings and what was changed, so Codex verifies fixes instead of
   rediscovering.
4. Round 3 still FAIL → **stop condition**: stop and surface the
   unresolved findings to Nat verbatim, with the round files' paths.

Claude never silently overrules a Critical/Important Codex finding. If
Claude believes a finding is wrong, that disagreement goes to Nat at the
gate (plan) or as a stop (final) — never self-adjudicated.

### Fallback (Codex unavailable or declined)

- Plan gate: today's self-review only (already in planning.md).
- Final review: today's Claude final-reviewer subagent on the most
  capable model (already in subagents.md).
- The Report's **Challenged by** line states it plainly, e.g.
  `Challenged by: Claude self-review (Codex unavailable)` — the bias
  check's absence is visible, never silent.

### Report line

New line in the `## Done:` report, after **How it was verified**:

```
**Challenged by:** Codex — plan: 2 rounds, final: 1 round (PASS)
```

or the fallback wording above.

## SKILL.md edit map (surgical)

| Location | Change |
|----------|--------|
| Phase 0 — Intake | New bullet: Codex check (probe + one-time setup ask) |
| Reference load-table | New row: Codex mode on (or undecided) → read `references/codex.md` at Plan and Test |
| Phase 2 — Plan | New step between self-review (4) and gate (5): Codex plan-challenge loop; gate summary states Codex verdict + rounds |
| Phase 4 — Test | Final reviewer paragraph: Codex challenge loop when Codex mode is on; Claude final-reviewer subagent as fallback |
| Stop conditions | New: Codex challenge still FAIL after 3 rounds |
| Phase 5 — Report | New **Challenged by** line in the report shape |

## Error handling

- `codex exec` non-zero exit or timeout → retry once; second failure →
  fall back to Claude review for that checkpoint and mark the Report line
  `(Codex errored — Claude fallback)`. Never blocks the ticket.
- Auth errors → surfaced with the one-time ask (see Setup check).
- Missing verdict line → one re-run, then treat as FAIL (see contract).

## Testing

Manual, on the next real tickets (the skill has no automated harness):
1. A code ticket with Codex installed — verify both checkpoints run,
   round files land in `.one2ten/briefs/`, Report carries the line.
2. An MCP-only ticket (n8n or Supabase) — verify live state is exported
   and Codex reviews the exported files.
3. A machine without Codex — verify the single intake ask, the fallback
   path, and the fallback Report wording.
