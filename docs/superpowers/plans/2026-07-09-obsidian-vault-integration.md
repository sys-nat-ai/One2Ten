# Obsidian Vault Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One2Ten reads the project's Obsidian index note for context and
keeps it updated after every ticket.

**Architecture:** New `references/vault.md` playbook (detection, note
lookup, template, update rules) + small hooks in SKILL.md (Phase 0
detection/read, Phase 5 update, load-table row). Plain file operations —
no MCP dependency. Spec:
`docs/superpowers/specs/2026-07-09-obsidian-vault-integration-design.md`.

**Tech Stack:** Markdown only (skill documentation).

## Global Constraints

- Vault steps activate ONLY when `.obsidian/` exists in an ancestor of the
  working project; otherwise the skill behaves exactly as today.
- Index note path: `<vault>/Projects/<project>.md`.
- Note updates are append/extend only; never delete or rewrite existing
  content beyond ticking a checkbox or correcting a stale asset entry.
- Credential names only in the note, never secret values.
- Tests are subagent probes (writing-skills TDD): RED baseline was run
  before this plan; GREEN probe after Task 3.

---

### Task 1: Create `references/vault.md`

**Files:**
- Create: `references/vault.md`

**Interfaces:**
- Produces: the playbook SKILL.md's hooks (Task 2) point at.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
# Vault Playbook — Obsidian second-brain integration

The working project may live inside an Obsidian vault (a second brain).
When it does, the vault's `Projects/` folder holds a per-project index
note: accumulated context flows in at Understand/Plan, durable knowledge
flows back at Report.

## Detection & lookup

- Walk up from the working project directory looking for `.obsidian/`.
  Found → vault mode; the vault root is that folder's parent. Not found →
  skip this playbook entirely.
- The index note is `<vault>/Projects/<project>.md` — `<project>` is the
  working folder's name, or the existing `Projects/` note whose title
  clearly matches this project. No match → create it at Phase 5 from the
  template below.

## Read (Understand / Plan)

- Read the note before planning. Key Assets, Conventions & Decisions, and
  Open Follow-ups are planning input: they seed exploration targets and
  the plan's Global Constraints.
- **Live state wins.** Verify the note's asset names/IDs against the real
  systems during Plan exploration — notes go stale. Track discrepancies
  and correct them in the note at Phase 5.

## Note template

```markdown
---
tags: [project]
status: active
---

# <Project Name>

**Systems:** <n8n / Make / Supabase / code / Vapi>
**Working dir:** <path relative to vault root>

## Key Assets
- <system>: <exact asset name (ID)>
- Credentials: <names only — NEVER secret values>

## Conventions & Decisions
- <YYYY-MM-DD> — <decision>

## Open Follow-ups
- [ ] <item> (<YYYY-MM-DD>, from <ticket title>)

## Ticket Log
- <YYYY-MM-DD> — **<ticket title>** — done. <one line: what changed; root
  cause if bug>
```

## Update (Phase 5, after composing the Report)

- Prepend a ticket-log line (newest first).
- Add new assets to Key Assets; correct entries proven stale during the
  run.
- Tick follow-ups this ticket resolved; add new ones from the Report's
  follow-ups section.
- Record decisions made during the ticket under Conventions & Decisions.
- Create the note from the template if it doesn't exist.
- Use `[[wikilinks]]` where a related vault note exists.

## Update discipline (vault hard rule)

- Append and extend only — never delete or rewrite existing note content
  beyond ticking a checkbox or correcting a stale asset entry.
- End the chat Report with a **Vault note updated** section quoting
  exactly the lines added or changed — the Report is the required
  highlight of the note edit.
- Credential names only, NEVER secret values — the vault may sync.
````

- [ ] **Step 2: Commit**

```bash
git add references/vault.md
git commit -m "feat: vault playbook - Obsidian index-note read/update rules"
```

### Task 2: SKILL.md hooks

**Files:**
- Modify: `SKILL.md` (Phase 0, load-table, Phase 5, description)

**Interfaces:**
- Consumes: `references/vault.md` (Task 1).

- [ ] **Step 1: Add Phase 0 vault check** — insert after the resume-check
  bullet:

```markdown
- **Vault check:** walk up from the working project for an `.obsidian/`
  directory. Inside an Obsidian vault → read `references/vault.md`, then
  read the project's index note (`Projects/<project>.md` in the vault) —
  its assets, conventions, and open follow-ups are Understand/Plan input.
```

- [ ] **Step 2: Add load-table row** — after the code/Vapi row:

```markdown
| Project is inside an Obsidian vault (`.obsidian/` in an ancestor) | `references/vault.md` |
```

- [ ] **Step 3: Add Phase 5 update step** — insert before the report
  shape block:

```markdown
In vault mode, also update the project's index note per
`references/vault.md`, and end the Report with a **Vault note updated**
section quoting exactly the lines added or changed.
```

- [ ] **Step 4: Extend the description frontmatter trigger** — append to
  the description: `Projects may live inside an Obsidian vault; the skill
  reads and maintains the vault's per-project index note.` ONLY if this
  keeps the description a when-to-use statement — do NOT describe the
  update workflow. (If it reads as workflow summary, skip this step.)

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "feat: vault hooks - detect vault, read index note, update at Report"
```

### Task 3: README mention

**Files:**
- Modify: `README.md` (Why bullets + repo layout)

- [ ] **Step 1: Add a Why bullet** after the production-safe bullet:

```markdown
- 🧠 **Second-brain aware** — if the project lives in an Obsidian vault,
  the skill reads the project's index note for context and keeps it
  updated after every ticket
```

- [ ] **Step 2: Add `vault.md` to the repo-layout block** after `code.md`:

```text
  vault.md                # 🧠 Obsidian vault playbook (index notes)
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: README - second-brain integration"
```

### Task 4: GREEN probe + install sync

- [ ] **Step 1: Re-run the RED baseline probe** (same prompt: vault
  project, populated `Projects/ClientX Booking App.md`, no `.one2ten/`)
  against the edited files. Expected: the narration reads the index note
  before planning, and Phase 5 updates it and quotes the change in the
  Report. If it fails, fix wording and re-probe.

- [ ] **Step 2: Sync the installed copy**

```powershell
Copy-Item "One2Ten\SKILL.md" "$env:USERPROFILE\.claude\skills\one2ten\SKILL.md" -Force
Copy-Item "One2Ten\references\*.md" "$env:USERPROFILE\.claude\skills\one2ten\references\" -Force
```

Expected: `references/` in the install dir now contains 7 files including
`vault.md`.
