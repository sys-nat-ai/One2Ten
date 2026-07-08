# One2Ten — Obsidian Vault Integration — Design

**Date:** 2026-07-09
**Owner:** Nat (AI Engineer)
**Status:** Approved (approval delegated after design review in chat)

## What this adds

Nat's projects all live inside an Obsidian vault (the second brain). One2Ten
gains two integrations with it:

1. **Read vault context** — during Understand/Plan, the skill reads the
   project's index note in the vault (`Projects/<project>.md`) for
   accumulated context: asset names/IDs, credential names, conventions,
   past decisions, open follow-ups.
2. **Maintain a per-project index note** — after the Report phase, the
   skill updates (or creates) that note: ticket log entry, changed assets,
   follow-ups, status. The vault stays current automatically.

Explicitly NOT in scope (per Nat's choices): separate per-ticket notes, and
moving plans/briefs/ledger out of `.one2ten/` into visible vault folders.
Working artifacts stay machine-state; the index note holds the durable
knowledge.

## Architecture

- New reference: `references/vault.md` — vault detection, note lookup,
  note template, update rules. Loads only when a vault is detected.
- `SKILL.md` hooks: vault detection in Phase 0, note read in
  Understand/Plan input, note update step in Phase 5, load-table row.
- No MCP/plugin dependency — vault notes are plain Markdown; existing file
  tools suffice.

## Behavior

### Detection (Phase 0)

Walk up from the working project directory looking for a `.obsidian/`
directory. Found → vault mode on; the vault root is that directory's
parent. Not found → every vault step silently skips (the skill is
unchanged for non-vault installs).

### Note lookup

`<vault>/Projects/<project>.md`, where `<project>` defaults to the working
folder's name; if no exact match, pick the `Projects/` note whose title
clearly matches the ticket's project. No match → no read; the note is
created at Phase 5.

### Read side (Understand / Plan)

- The note's Key Assets, Conventions & Decisions, and Open Follow-ups are
  planning input (they seed exploration and Global Constraints).
- **Live state wins over the note.** Asset IDs from the note are verified
  against live systems during Plan exploration; the note can be stale.
  Discrepancies get corrected in the note at Phase 5.

### The index note (template)

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
- <date> — <decision>

## Open Follow-ups
- [ ] <item> (<date>, from <ticket title>)

## Ticket Log
- <YYYY-MM-DD> — **<ticket title>** — done. <one line: what changed;
  root cause if bug> <[[wikilinks]] to related notes where they exist>
```

### Write side (Phase 5)

After composing the chat Report:

- Prepend the ticket-log line (newest first).
- Add new assets to Key Assets; correct entries proven stale during the
  run.
- Tick follow-ups this ticket resolved; add new ones from the Report's
  follow-ups section.
- Record noteworthy decisions made during the ticket.
- Create the note from the template if it doesn't exist.

**Update discipline (vault hard rule):** append and extend only — never
delete or rewrite existing note content beyond ticking a checkbox or
correcting a stale asset ID; the Report gains a final **Vault note
updated** section quoting exactly the lines added or changed, so the
Report itself is the required highlight. Credential names only, never
secret values (the vault may sync).

## Testing (writing-skills TDD)

- **RED:** baseline probe — agent with the current skill, project inside a
  vault with a populated `Projects/<project>.md`; observe whether it reads
  or updates the note (expected: no).
- **GREEN:** same probe against the edited skill — the narration must
  read the note before planning, and Phase 5 must update it and quote the
  change in the Report.
