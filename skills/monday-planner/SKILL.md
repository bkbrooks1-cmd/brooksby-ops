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

Required keys: `notion.week_plans_page`, `notion.tasks_db`, `notion.leads_db`, `notion.meetings_db`, `notion.engagements_db`, `notion.internal_engagements`, `email.monitored_addresses`, `voice.style_guide_path`, `firewall.no_connector_accounts`.

`notion.week_plans_page` is where the finished plan lands. It is the same page `friday-wrap` writes wraps to, so plans and wraps sit together in date order. If that key is absent, fall back to `notion.home_page` and say so in the output.

Optional: a `content` block (see config.example.json). If present, the plan carries the standing content deliverables and the roadmap slot (section 6 below); `content.default_post_count` is used if set. If absent, skip everything content-related — no error, no mention.

**Storage.** The content section names no path. It calls named operations in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`, which holds one procedure per backend.

## Engagement routing

Every task this skill creates carries an engagement. Never write a null Engagement. The rule, the four internal buckets, and how to pick between them live in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md` — read it before creating tasks.

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

Create a sub-page of the Weekly Plans and wraps page (page_id `{notion.week_plans_page}`), titled "Week plan — YYYY-MM-DD" (the Monday date). Never create it under `notion.home_page` — the plan belongs alongside the week wraps, not at the root. Structure, in order:

1. **Sources missed** — only if a source failed. One line naming the source and what is therefore not covered.
2. **Week at a glance** — meetings by day, each with: time, title, engagement (if known), and prep status (prep task exists / done / none needed).
3. **Top priorities** — three to five, drawn from due tasks, overdue items, and meeting prep. Outcome phrasing, not activity phrasing.
4. **Tasks due this week** — linked task list by day.
5. **Pipeline** — leads needing action this week, each with its next action. Flag leads missing a next action.
6. **Content** — only if a `content` block exists in config. Division of labor: Notion owns tasks and the calendar; the content system owns drafts and research. First, read **the theme map** and open this section with the **roadmap slot**: one line naming the theme the forward map has queued for this week (series + working theme), so the week's content aims at something. Then list the standing deliverables, each linked to where its draft will live:
   - "Newsletter #N" — N = the next issue number from **list drafts** (ask if unclear).
   - "LinkedIn posts xN" — N derived posts from the newsletter, where N = `content.default_post_count` (default 3); the user sets the actual count at draft time.
   - "Metrics log" — the weekly numbers entry.

   Then run **list drafts** again for anything still at `status: drafted` from prior weeks and flag it here as unpublished. Omit this section entirely when no `content` block exists.
7. **Meeting capture** — only if Granola is connected: run the capture routine (canonical steps live in the capture skill) for last week's meetings. For each meeting not already in the Meetings DB, propose a meeting page with summary-based minutes and its action items, infer the engagement (confirm if unsure), and create on confirmation. Fold the resulting action items into "Tasks due this week" and "Carryover" above. Omit this section entirely when Granola is not connected.
8. **Carryover** — overdue open tasks rolled in from prior weeks.

## Output 2: prep tasks

For each meeting this week that plausibly needs preparation (client meetings, anything with an external attendee or an agenda) and has no existing prep task: create a Task row (data source `collection://{notion.tasks_db}`) with Type = Prep, Due date = the day before the meeting, Source = Planning, and an Engagement.

Link the identified client engagement. If none is identifiable, route by the bucket rule rather than leaving it blank: a prep task for a non-client meeting goes to Business Development when there is a named opportunity behind it, or Networking when it is relationship maintenance. Never null. If more than five prep tasks would be created, list them and ask before creating.

If a `content` block exists in config, also create the three standing content Tasks for the week — "Newsletter #N", "LinkedIn posts xN" (N = `content.default_post_count`, default 3), "Metrics log" — each with Type = Deliverable, Source = Planning, Due date = this Friday, Engagement = the Marketing Content bucket (`notion.internal_engagements.marketing_content`), and a link to where the draft will live in the task body. Dedupe first: if an open task with the same name already exists (a carryover), roll it instead of creating a twin.

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
- Never write a task with a null Engagement. Client work links to its client row; everything else routes to an internal bucket per `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md`.
- Accounts listed in `firewall.no_connector_accounts` are never connected. Anything bound for them is a draft the user sends.
- If both calendar and mail fail, deliver what Notion alone supports and say plainly that the plan is running on one engine.
