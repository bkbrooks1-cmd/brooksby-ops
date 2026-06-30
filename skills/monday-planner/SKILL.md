---
name: monday-planner
description: Generate the user's weekly plan in Notion and draft the Monday weekly email. Use whenever the user says "plan my week", "run the Monday planner", "build my week plan", "what does my week look like", or when the scheduled Monday-morning planning task fires. Also use mid-week when the user asks to re-plan after new meetings or priorities land. If the user asks anything about organizing their upcoming week, reach for this skill rather than improvising.
---

# Monday planner

Build the week plan from every connected source, write it to Notion, and draft the weekly email for review. The user runs solo; this plan is how the week starts. A partial plan delivered on time beats a perfect plan that never generates.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path. (Onboarding writes this file to the folder the user chose at setup.)

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before running the planner."

Required keys: `notion.home_page`, `notion.tasks_db`, `notion.leads_db`, `notion.meetings_db`, `email.monitored_addresses`, `voice.style_guide_path`, `firewall.no_connector_accounts`.

## Voice

Everything written here is in the user's voice: plain, direct, metrics-first. If `voice.style_guide_path` points to a style guide, apply its rules to all prose, especially the weekly email. If it is empty or its file is unavailable, use a neutral plain-professional voice — never stop over a missing style guide. No buzzwords, no filler, no em dashes.

## Sources to read

Read every connected source below (Granola is optional). If a source cannot be read (expired token, connector down), do not stop and do not silently omit it: name it in a "Sources missed" line at the top of the plan and keep going.

1. **Google Calendar** (connector): all events from today through Sunday, all calendars. This is the single calendar hub; it includes Calendly bookings and invites forwarded from the addresses in `email.monitored_addresses`.
2. **Gmail** (connector): last 7 days. Look for unresolved items: unread or flagged threads, direct questions, anything addressed to any address in `email.monitored_addresses` (all forward into this inbox). An item is unresolved if it plainly asks for something and no reply exists.
3. **Notion Tasks**: data source `collection://{notion.tasks_db}`. Open tasks = Status is not Done. Note anything overdue.
4. **Notion Leads**: data source `collection://{notion.leads_db}`. Rows with Next action date on or before this Friday, plus any row with an empty Next action (that emptiness is itself a flag; the pipeline rule is Next action always filled).
5. **Granola** (connector, optional — skip cleanly if not connected): meetings from the past 7 days. Cross-check against the Notion Meetings DB (`collection://{notion.meetings_db}`); any Granola meeting without a Meetings page is an uncaptured meeting worth flagging.

If a project folder is mounted in the session, also scan active planning documents for commitments with dates.

## Output 1: the week plan page in Notion

Create a sub-page of the home page (page_id `{notion.home_page}`), titled "Week plan — YYYY-MM-DD" (the Monday date). Structure, in order:

1. **Sources missed** — only if a source failed. One line naming the source and what is therefore not covered.
2. **Week at a glance** — meetings by day, each with: time, title, engagement (if known), and prep status (prep task exists / done / none needed).
3. **Top priorities** — three to five, drawn from due tasks, overdue items, and meeting prep. Outcome phrasing, not activity phrasing.
4. **Tasks due this week** — linked task list by day.
5. **Pipeline** — leads needing action this week, each with its next action. Flag leads missing a next action.
6. **Uncaptured meetings** — only if Granola is connected: meetings from last week with no Notion Meetings page, flagged so the user can add them. (Omit this section entirely when Granola is not connected.)
7. **Carryover** — overdue open tasks rolled in from prior weeks.

## Output 2: prep tasks

For each meeting this week that plausibly needs preparation (client meetings, anything with an external attendee or an agenda) and has no existing prep task: create a Task row (data source `collection://{notion.tasks_db}`) with Type = Prep, Due date = the day before the meeting, Source = Planning, Engagement linked when identifiable. If more than five prep tasks would be created, list them and ask before creating.

## Output 3: weekly email draft(s)

Draft only — never send. The user reviews and sends from their own mailbox.

Decide which emails to draft from config:

- **No engagement is flagged for a weekly email** (none has a `weekly_email` block, or `engagements` is empty): produce one **generic weekly summary email** in the user's voice, using the default template at `${CLAUDE_PLUGIN_ROOT}/templates/weekly_email_TEMPLATE.html` (or an equivalent neutral format if that file is unavailable). Leave the recipient line blank for the user to fill. Save it as an HTML file in the user's chosen project folder, named `Weekly_Email_<M_D>_v1.html` (M_D = the Monday date; bump `_v2`, `_v3` on re-runs).
- **One or more engagements are flagged** (Engagements row has `Weekly report` = true and the engagement has a `weekly_email` block in config): draft one email per flagged engagement, using that engagement's own `weekly_email` settings — its `template_path`, `to`, `cc`, `subject_format`, and `save_to`. Each engagement carries its own copied-and-customized template; the planner just fills it. See `references/weekly-email.md` for the config shape and how to set up an engagement template.

Default generic structure (and the baseline every engagement template starts from), in order:

- Header block: **To** (from config, or blank), **Cc** (from config, or the user from `user.name` / `user.company`), **Subject** (from `subject_format`, or "M/D Weekly Check-in - <week theme>").
- One-line greeting, then a short framing note if relevant (short week, OOO, etc.).
- **Where we stand** — 2 to 4 bullets on status and momentum.
- **What is happening this week** — meetings, each with day, time, location, and organizer.
- **What I need from you** — named asks, one bullet per person.
- **On my side this week** — the user's own commitments and any OOO.
- One-line sign-off, then the user's name from `voice.name` (fall back to `user.name`).

Apply the voice rules above. If `voice.style_guide_path` is empty or its file is unavailable, write in a neutral plain-professional voice — do not stop.

## Hard rules

- Never send email. Never post anything. Drafts only; the user sends.
- Never delete or close existing tasks during planning. Carryover gets rolled, not purged.
- Accounts listed in `firewall.no_connector_accounts` are never connected. Anything bound for them is a draft the user sends.
- If both calendar and mail fail, deliver what Notion alone supports and say plainly that the plan is running on one engine.
