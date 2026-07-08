# Planning Reference — plan document format

Plans go to `.one2ten/plans/YYYY-MM-DD-<ticket-slug>.md` in the working
project. Write for an implementer with zero context: they see only their
own task, so every task must be self-sufficient.

## Plan header

```markdown
# <Ticket title> — Plan

**Goal:** <one sentence>
**Approach:** <2–3 sentences>
**Systems touched:** <n8n / Make / Supabase / code — with the exact assets>

## Global Constraints
<exact values from the ticket, one per line: URLs, IDs, credential names,
formats, copy text. Every task implicitly includes these.>

## Acceptance Criteria
<numbered, testable criteria from the Understand phase. The Test phase
verifies exactly these — if it is not listed here, it will not be tested.>
```

## Task structure

````markdown
### Task N: <name>

**Targets:**
- n8n: <workflow name> (ID: <id>)   ← exact, from exploration, never guessed
- Supabase: <table / function name>
- Files: `exact/path/to/file.ts`

**Interfaces:**
- Consumes: <what this task uses from earlier tasks — exact names, field shapes>
- Produces: <what later tasks rely on — exact names, signatures, payload shapes>

- [ ] **Step 1:** <one action, with the actual content — see content rule>
- [ ] **Step 2:** ...
- [ ] **Verify:** <exact tool call or command> → expected: <exact result>
````

## Rules

- **Bite-sized:** each step is one action (2–5 minutes). Each task is the
  smallest unit with its own Verify step — split only where a reviewer
  could reject one task while approving its neighbor.
- **Content rule — the plan contains the real thing:**
  - n8n task → the actual node configuration values and expressions.
  - Supabase task → the actual SQL.
  - Make task → the actual module parameters.
  - Code task → the actual code (and the failing test first — TDD).
- **No placeholders.** These are plan failures, never write them: "TBD",
  "TODO", "add error handling", "handle edge cases", "similar to Task N"
  (repeat the content instead), steps that describe without showing.
- **Destructive actions:** any step that deletes, deactivates, drops, or
  overwrites a production asset gets a `⚠️ DESTRUCTIVE` marker and is
  called out in the plan summary at the approval gate.

## Self-review (before presenting the plan)

1. **Coverage:** every ticket requirement maps to a task, and every
   acceptance criterion is verifiable by a Verify step or the Test phase.
   List gaps and fix.
2. **Placeholder scan:** search the plan for the failure patterns above.
3. **Consistency:** names, IDs, signatures, and field shapes used in later
   tasks match what earlier tasks define — exactly.

Fix findings inline, then present the summary + path at the gate.
