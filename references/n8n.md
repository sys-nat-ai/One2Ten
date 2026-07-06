# n8n Playbook

## Explore first (Understand / Plan phases)

- `search_workflows` → find the workflow; `get_workflow_details` → fetch
  the FULL current JSON before planning any edit.
- Bug evidence: `search_executions` (filter by workflow, status=error) →
  `get_execution` on the failing run. Read the actual node error and input
  data — root cause comes from here, not from guessing.

## Before building

- `get_sdk_reference` — mandatory before writing any workflow SDK code; do
  not write from memory.
- `get_workflow_best_practices` — once per relevant technique (use
  `technique="list"` if unsure which apply).
- `search_nodes` for every service and utility node → `get_node_types` with
  ALL node IDs including discriminators (resource/operation/mode). Never
  guess parameter names.
- Parameters annotated `@searchListMethod` / `@loadOptionsMethod` (channel
  pickers, model lists, sheet tabs): resolve real values with
  `explore_node_resources` + a credential ID from `list_credentials`.
  Never invent IDs.
- Writing expressions or Code nodes? Use the `n8n-expression-syntax` and
  `n8n-code-javascript` skills.

## Build

- New workflow: `create_workflow_from_code`. Existing: `update_workflow` —
  only after `get_workflow_details` fetched the current state, and after
  saving that JSON to `.one2ten/briefs/<workflow-id>-before.json` as the
  rollback copy.
- Validate before anything ships: `validate_node_config` per configured
  node, `validate_workflow` on the whole thing.

## Test (evidence required)

- `prepare_test_pin_data` → `test_workflow`; inspect the run with
  `get_execution` — check each node's output, not just overall status.
- Only after a passing test: `publish_workflow`.

## Safety

- `unpublish_workflow`, `archive_workflow`, or replacing a live workflow's
  trigger: approval-gated — ask the human first.
- Prefer adding nodes/branches over rewiring existing paths that other
  flows depend on.
