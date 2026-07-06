# Supabase Playbook

## Explore first (Understand / Plan phases)

- `list_projects` / `get_project` → confirm which project; `list_tables`,
  `list_migrations`, `list_extensions` before ANY schema plan.
- Edge functions: `list_edge_functions` / `get_edge_function` (fetch
  current source before planning an edit).
- Bug evidence: `get_logs` (per service: api, postgres, edge-function…) and
  `get_advisors` — read these BEFORE changing anything on a bug ticket.

## Build

- **Schema/DDL only via `apply_migration`** with a descriptive migration
  name. Never DDL through `execute_sql` — it bypasses migration history.
- Data fixes and verification queries: `execute_sql`.
- Edge functions: `deploy_edge_function`; save the previous source (from
  `get_edge_function`) to `.one2ten/briefs/` as rollback first.
- If the project's app consumes generated types, refresh with
  `generate_typescript_types` after schema changes.

## Risky changes — use a branch

For migrations that could break production: `create_branch` → apply and
verify on the branch → `merge_branch`. Abandon with `reset_branch` /
`delete_branch`. (`create_branch` may require `confirm_cost` first.)

## Test (evidence required)

- `execute_sql` assertions with the expected rows stated up front
  ("expect 3 rows where status='active'").
- `get_logs` after deploys; `get_advisors` after schema changes — fix new
  security/performance advisories or report them explicitly in Phase 5.

## Safety (approval-gated — ask the human first)

- `DROP`, `TRUNCATE`, `DELETE` without `WHERE`, disabling RLS.
- `pause_project` / `restore_project`.
