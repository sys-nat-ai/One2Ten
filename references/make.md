# Make Playbook

## Explore first (Understand / Plan phases)

- `scenarios_list` → find the scenario; `scenarios_get` → fetch the full
  blueprint before planning any edit.
- Bug evidence: `executions_list` (or `executions_get` for a known run) →
  `executions_get-detail`. Read the failing module's input/output — root
  cause comes from here.
- `connections_list` → confirm the needed connections/credentials exist
  before planning around them; `connection-metadata_get` for details. If a
  connection is missing, that is a stop condition — name it and ask.

## Build

- Find the right modules: `apps_recommend` / `apps_list` →
  `app-modules_list` → `app-module_get` + `app_documentation_get` for exact
  parameter shapes. Never guess module parameters.
- Save the current blueprint to
  `.one2ten/briefs/<scenario-id>-before.json` as rollback before
  `scenarios_update` on anything live.
- Validate before create/update: `validate_blueprint_schema`,
  `validate_module_configuration` per module; scheduling changes via
  `validate_scheduling_schema`; webhooks via `hooks_create`/`hooks_update`
  checked with `validate_hook_configuration`.
- Data stores: `data-stores_*` and `data-structures_*` (use
  `data-structures_generate` from a sample payload rather than hand-writing
  specs).

## Test (evidence required)

- `scenarios_run` → `executions_get-detail` on that run. Check every
  module's output bundle, not just the scenario-level status.

## Safety

- `scenarios_deactivate` and `scenarios_delete` are approval-gated — ask
  the human first.
- `scenarios_activate` for a brand-new scenario only after a passing test
  run.
