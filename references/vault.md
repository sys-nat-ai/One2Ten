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
