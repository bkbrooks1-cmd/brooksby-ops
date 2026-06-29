---
name: daily-checkin
description: Brian's daily check-in - today's meetings with prep status, email triage into tasks, deliverable check-off, and plan updates. Use whenever Brian says "check in", "what's on my plate", "what does today look like", "morning rundown", "catch me up", or asks for a status of his day or tasks at any point. If Brian asks what he should be doing right now, use this skill.
---

# Daily check-in

The two-minute view that keeps the plan honest between Mondays. Read fast, show what matters, update only what the user confirms.

## Config

This skill reads instance values from `solo-os-config.json` at the Solopreneur OS project root (`Projects/Solopreneur OS/solo-os-config.json`). Load it first.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before running the check-in."

Required keys: `notion.tasks_db`, `notion.leads_db`, `notion.agent_ideas_db`, `email.monitored_addresses`, `firewall.no_connector_accounts`.

## Sources to read

Flag any source that fails; never silently omit it.

1. **Google Calendar** (connector): today and tomorrow, all calendars.
2. **Gmail** (connector): since the last check-in (default: last 24 hours). Unread or flagged threads and direct asks, including mail to any address in `email.monitored_addresses` (all forward here).
3. **Notion Tasks**: data source `collection://{notion.tasks_db}`. Due today, overdue, and In progress.

## Output: the today view, in chat

Keep it tight. Four blocks, skip any that are empty:

1. **Today's meetings** — time, title, prep status. Tomorrow's meetings that need prep today get one line each.
2. **Due and overdue** — tasks due today, then overdue, each with engagement. Overdue items get a suggested move: do today, reschedule (propose a date), or flag for Friday wrap.
3. **From email** — new items that look like work. For each, propose a task (name, due date, engagement). Create rows only after the user confirms. Mark Source = Email.
4. **Deliverables** — anything produced in the current session that maps to an open task; offer to mark it Done.

## Update rules

- Create tasks only with confirmation. Batch the proposals so the user can say "yes to all" or pick.
- Never move a due date or close a task Claude did not create without an explicit yes.
- "Posted", "done", "sent" from the user about a named item = mark the matching task Done without re-asking.
- New leads spotted in email: offer a row in Leads (`collection://{notion.leads_db}`) with Stage = Lead, Source = Email inquiry, and a drafted next action.
- Agent ideas the user voices mid-check-in go to Agent Ideas (`collection://{notion.agent_ideas_db}`) immediately; low risk, no confirmation needed, show the row.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Accounts listed in `firewall.no_connector_accounts` are never connected.
- A failed source means a smaller view, not a silent one: open with "Gmail unavailable" (or whichever) and continue.
