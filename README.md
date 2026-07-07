<div align="center">

# ðŸŽ¯ One2Ten

### Ticket in. Verified work out.

**An end-to-end ticket workflow skill for Claude Code** â€” runs any task
ticket from intake to an evidence-backed completion report, across n8n,
Make, Supabase, and code.

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-cc785c?style=for-the-badge&logo=anthropic&logoColor=white)](https://claude.com/claude-code)
[![n8n](https://img.shields.io/badge/n8n-workflows-ea4b71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io)
[![Make](https://img.shields.io/badge/Make-scenarios-6d00cc?style=for-the-badge&logo=make&logoColor=white)](https://make.com)
[![Supabase](https://img.shields.io/badge/Supabase-database-3fcf8e?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com)

`Understand â†’ Plan â†’ Build â†’ Test â†’ Report`
**One approval gate. Zero unverified claims.**

</div>

---

## âœ¨ Why

AI-engineer tickets rarely live in one place â€” a bug might span an n8n
workflow, a Supabase table, and a voice agent. One2Ten gives Claude Code a
single disciplined process for all of it:

- ðŸ” **Evidence before fixes** â€” bugs get root-cause proof from real
  executions and logs before any plan is written
- ðŸ“‹ **Plans with no hand-waving** â€” every step contains the actual SQL,
  node config, or code; "add error handling" is a banned phrase
- ðŸ¤– **Multi-agent build** â€” fresh implementer subagent per task, reviewer
  subagent after each, fix loops until clean
- ðŸ§¾ **Receipts required** â€” nothing is reported "done" without the command
  that proves it and its observed output
- ðŸ›¡ï¸ **Production-safe** â€” live assets are fetched before edits, rollback
  copies saved, destructive actions always ask first

## ðŸ”„ The Flow

```mermaid
flowchart LR
    A["ðŸ§  Understand<br/><sub>fetch ticket Â· classify Â·<br/>root-cause bugs first</sub>"] --> B["ðŸ“‹ Plan<br/><sub>explore live state Â·<br/>task-by-task plan</sub>"]
    B --> G{{"ðŸš¦ Your approval<br/><sub>the only stop</sub>"}}
    G --> C["ðŸ”¨ Build<br/><sub>implementer + reviewer<br/>subagents per task</sub>"]
    C --> D["ðŸ§ª Test<br/><sub>end-to-end vs acceptance<br/>criteria Â· evidence rule</sub>"]
    D --> E["ðŸ“¬ Report<br/><sub>what changed Â· proof Â·<br/>root cause Â· follow-ups</sub>"]
```

| Phase | What happens |
|:-----:|--------------|
| ðŸ§  **Understand** | Fetches the ticket (Zoho Projects MCP) or restates your chat brief; classifies **feature / task / bug**. Bugs get evidence first: failing executions and logs pulled, root cause *demonstrated* â€” never guessed. |
| ðŸ“‹ **Plan** | Explores the live state of every affected system, then writes a task-by-task plan to `.one2ten/plans/` with real node configs, SQL, and code in every step. Destructive steps flagged âš ï¸. **Stops once for your approval.** |
| ðŸ”¨ **Build** | Subagent-driven: fresh implementer per task, reviewer after each (git diff for code Â· live MCP state for platform work), fix loops, and a progress ledger that survives session crashes. |
| ðŸ§ª **Test** | Verifies the *ticket's* acceptance criteria end-to-end â€” real workflow runs, real queries, real app flows. Every "it works" carries proof. |
| ðŸ“¬ **Report** | Chat summary: exact asset names/IDs changed, verification evidence, one-sentence root cause (bugs), follow-ups. |

## ðŸš€ Install

**Windows (PowerShell)**
```powershell
git clone https://github.com/sys-nat-ai/One2Ten.git one2ten-skill
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills\one2ten\references"
Copy-Item "one2ten-skill\SKILL.md" "$env:USERPROFILE\.claude\skills\one2ten\"
Copy-Item "one2ten-skill\references\*.md" "$env:USERPROFILE\.claude\skills\one2ten\references\"
```

**macOS / Linux**
```bash
git clone https://github.com/sys-nat-ai/One2Ten.git one2ten-skill
mkdir -p ~/.claude/skills/one2ten
cp one2ten-skill/SKILL.md ~/.claude/skills/one2ten/
cp -r one2ten-skill/references ~/.claude/skills/one2ten/
```

> ðŸ’¡ Prefer a per-project install? Drop `SKILL.md` + `references/` into
> `<project>/.claude/skills/one2ten/` instead. New Claude Code sessions
> pick it up automatically.

## ðŸŽ® Use

```text
/one2ten <your ticket>
```

```text
/one2ten Fix the lead-sync workflow in n8n â€” failing since yesterday
/one2ten Zoho task "Add SMS reminders" in the ClientX project
```

â€¦or just paste a ticket / bug report and ask Claude to handle it end to end.

**Tips**

- ðŸ“‚ Run it from the project folder the ticket belongs to â€” plans, briefs,
  and the ledger land in that project's `.one2ten/` directory
- ðŸ” Session died mid-ticket? Re-invoke it â€” the `.one2ten/progress.md`
  ledger prevents redoing completed tasks
- â¸ï¸ After plan approval it only interrupts for: destructive actions,
  missing credentials, genuine ambiguity, or an unresolvable blocker

## ðŸ—‚ï¸ Repo layout

```text
SKILL.md                  # ðŸŽ¯ the orchestrator: phases, gate, rules
references/
  planning.md             # ðŸ“‹ plan format â€” no-placeholder rules
  subagents.md            # ðŸ¤– dispatch contracts, review modes, ledger
  n8n.md                  # ðŸ”§ n8n MCP playbook
  make.md                 # ðŸ”§ Make MCP playbook
  supabase.md             # ðŸ”§ Supabase MCP playbook
  code.md                 # ðŸ”§ code / apps / Vapi playbook
docs/superpowers/         # ðŸ“œ design spec + implementation plan (history)
```

## ðŸ“¦ Requirements

- **Claude Code** with subagent support (the `Agent` tool) for the Build phase
- The **MCPs your tickets touch** connected â€” n8n, Make, Supabase, Zoho
  Projects, Vapi (only the ones you actually use)

## ðŸ™ Credits

Workflow patterns adapted from the
[Superpowers](https://github.com/obra/superpowers) skill family â€”
*writing-plans*, *subagent-driven-development*, *systematic-debugging*,
*verification-before-completion* â€” tailored for MCP-based automation work
where changes live outside git.

<div align="center">

---

Made with â˜• by **Nat**

</div>
