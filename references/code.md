# Code / Apps / Vapi Playbook

## Features and tasks — TDD

- Per task: write the failing test → run it, confirm it fails for the right
  reason → minimal implementation → run it, confirm pass → commit. Small,
  frequent commits.
- Follow the repo's existing patterns (structure, naming, test style, error
  handling). Read neighboring code before writing new code.
- Multi-commit changes go on a branch, not main.

## Bugs — systematic debugging

1. Reproduce the failure and capture the actual error output.
2. Read the error. Trace to the real cause — form a hypothesis and prove it
   with evidence (logs, a targeted test, a debugger print).
3. Only then fix. The fix ships with a regression test that fails without
   it.

No fix is proposed on a guess. "I changed X and it seems fine" is not root
cause.

## Verification

- Run the project's real test command (check package.json / pyproject /
  Makefile for what it is) — full relevant suite, not just the new test.
- Exercise the changed flow end-to-end: run the app/server and drive the
  actual behavior the ticket asked for. Unit tests alone are not Phase 4
  evidence.

## Vapi voice agents

- `list_assistants` / `get_assistant` — fetch current config before any
  edit; save it to `.one2ten/briefs/<assistant-id>-before.json` as
  rollback.
- `update_assistant` on a live agent only after the rollback copy exists.
- Tools: `create_tool` / `update_tool` / `get_tool`; phone numbers via
  `list_phone_numbers` / `get_phone_number`.
- Verify with a real test call where feasible: `create_call` → `get_call`
  (check transcript/outcome). If a test call is not feasible, report
  config-level verification honestly — do not claim behavioral testing.

## Git hygiene

- Never `push --force`, `reset --hard`, or checkout-discard without
  approval.
- Commit messages state what changed and why, per-task.
