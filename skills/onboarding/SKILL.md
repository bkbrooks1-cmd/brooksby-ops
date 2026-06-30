---
name: onboarding
description: Stand up the Solopreneur OS for a new user from an empty Notion workspace. Use when someone says "set up the OS", "onboard me", "first-time setup", "get me started", "install the Solopreneur OS", or when any other skill reports that solo-os-config.json is missing. Provisions the six Notion databases, captures their IDs into config, connects Gmail/Calendar/Notion, and verifies the install with a check-in.
---

# Onboarding

Take a new user from nothing to a working daily check-in. This skill builds the six Notion databases the OS runs on, writes their IDs into `solo-os-config.json`, walks through connecting the tools, and verifies the whole thing before handing off.

Run it once per install. It is the make-or-break setup flow: if it works unattended, the OS travels to a new person; if it doesn't, nothing else matters.

## Before you start

Confirm the prerequisites in plain language. The user needs:

1. The **Notion** connector enabled in Cowork.
2. The **Gmail** and **Google Calendar** connectors enabled (the daily skills read both).
3. One Notion page they will let the OS use as its home — new or existing — shared with the Notion integration so it can create databases under it (in Notion: open the page → ••• menu → Connections → add the integration).
4. A Cowork project folder for the OS, where `solo-os-config.json` will be written.

If a connector is missing, stop and tell the user exactly which one to enable in Settings, then resume. Do not proceed past a missing Notion connector — every step below depends on it.

## Step 1 — Identify the workspace and home page

Call Notion `fetch` with id `"self"` to confirm the connected workspace and the user's identity. State the workspace name back to the user so they know where the databases will land.

Ask the user for the **home page**: the parent under which the six databases will be created. Accept a Notion URL or let them name a page to search for. Resolve it to a page ID. This ID becomes `notion.home_page` in config. If they have no page yet, create one titled with their company or "Solopreneur OS" and use that.

Then confirm the **project folder**: ask the user which Cowork project folder the OS should live in. That folder is where `solo-os-config.json` will be written in Step 4. Do not assume a folder — confirm it explicitly, and create it if it does not exist.

## Step 1.5 — Verify Notion access before building

Before creating anything real, confirm the integration can write under the home page. Create one throwaway database under it (a single-column "Access check") and confirm it returns an ID, then trash it. If creation is denied, stop and tell the user in plain language: open the home page in Notion, click the ••• menu, choose Connections, add the integration, then resume. This catches the most common setup failure — the integration not being granted page access — before any real work begins.

## Step 2 — Provision the six databases

The exact creation procedure is below and is self-contained — you do not need any external file to run it. (When installed as a plugin, `${CLAUDE_PLUGIN_ROOT}/skills/onboarding/references/notion-schema.md` carries extra rationale and genericization notes, but reading it is optional.)

The databases cross-link with two-way relations, and a relation can only point at a database that already exists. So **create them in this order**, capturing each returned data-source ID before moving on. Each two-way relation is declared inline with `RELATION('<target-ds-id>', DUAL '<mirror name>')`, which auto-creates the correctly named mirror property on the target — do not create mirrors by hand.

Use Notion `create-database` with `parent` = the home page ID for each. Substitute the captured IDs where shown.

**1. Engagements** (no outgoing relations — its relation properties arrive later as mirrors):
```
CREATE TABLE ("Client" TITLE, "Status" SELECT('Proposal':yellow, 'Active':green, 'Paused':orange, 'Closed':gray), "Billing model" SELECT('T&M':blue, 'Fixed':purple, 'Retainer':green), "Rate" NUMBER FORMAT 'dollar', "Start date" DATE, "Key contacts" RICH_TEXT, "Place" RICH_TEXT, "Weekly report" CHECKBOX COMMENT 'Drives the weekly-report agent')
```
Capture as `ENG_ID` → config `notion.engagements_db`.

**2. Meetings** (relates to Engagements):
```
CREATE TABLE ("Name" TITLE, "Date" DATE, "Attendees" RICH_TEXT, "Granola link" URL, "Engagement" RELATION('ENG_ID', DUAL 'Meetings'))
```
Capture as `MTG_ID` → config `notion.meetings_db`. Auto-creates `Meetings` on Engagements.

**3. Tasks** (relates to Engagements and Meetings):
```
CREATE TABLE ("Name" TITLE, "Status" SELECT('To do':red, 'In progress':yellow, 'Waiting':orange, 'Done':green), "Type" SELECT('Deliverable':blue, 'Prep':purple, 'Follow-up':yellow, 'Admin':gray, 'Networking':red), "Priority" SELECT('P1':red, 'P2':yellow, 'P3':gray), "Source" SELECT('Meeting':blue, 'Email':purple, 'Planning':green, 'Ad hoc':gray, 'Owner':brown), "Due date" DATE, "Engagement" RELATION('ENG_ID', DUAL 'Tasks'), "Meeting" RELATION('MTG_ID', DUAL 'Action items'))
```
Capture as `TASK_ID` → config `notion.tasks_db`. Auto-creates `Tasks` on Engagements and `Action items` on Meetings.

**4. Leads & Opportunities** (relates to Engagements):
```
CREATE TABLE ("Name" TITLE, "Company" RICH_TEXT, "Stage" SELECT('Lead':blue, 'Qualified':yellow, 'Proposal':orange, 'Won':green, 'Lost':gray), "Source" SELECT('LinkedIn':blue, 'Email inquiry':purple, 'Referral':green, 'Conversation':yellow, 'Calendly':orange), "Estimated value" NUMBER FORMAT 'dollar', "Next action" RICH_TEXT, "Next action date" DATE, "Notes" RICH_TEXT, "Converted to" RELATION('ENG_ID', DUAL 'Source lead'))
```
Capture as `LEAD_ID` → config `notion.leads_db`. Auto-creates `Source lead` on Engagements.

**5. Content Calendar** (standalone):
```
CREATE TABLE ("Title" TITLE, "Status" SELECT('Idea':gray, 'Drafted':yellow, 'Scheduled':orange, 'Posted':green), "Series" SELECT('Standalone':gray), "Post date" DATE, "Performance notes" RICH_TEXT)
```
Capture as `CONTENT_ID` → config `notion.content_db`.

**6. Agent Ideas** (standalone):
```
CREATE TABLE ("Idea" TITLE, "Status" SELECT('Backlog':gray, 'Next':yellow, 'Built':green, 'Dropped':red), "Value" SELECT('High':green, 'Medium':yellow, 'Low':gray), "Effort" SELECT('S':green, 'M':yellow, 'L':red), "Problem it solves" RICH_TEXT, "Trigger and data" RICH_TEXT)
```
Capture as `IDEAS_ID` → config `notion.agent_ideas_db`.

Known limitation: Notion's `place` property type cannot be created via DDL, so Engagements `Place` is provisioned as text. If the user wants a true location field, they convert it in the Notion UI after setup. Note this and move on — it is not setup-blocking.

## Step 2.5 — Build the home dashboard

Give the user a working cockpit on the home page instead of bare databases. Use Notion `create-view` with `parent_page_id` = the home page ID and `data_source_id` = the relevant captured ID. Each call appends a linked view block to the home page. Create these six, in order:

1. **Open tasks** — `data_source_id` = Tasks, type `table`, configure: `FILTER "Status" != "Done" SORT BY "Due date" ASC SHOW "Name", "Status", "Priority", "Due date", "Engagement"`
2. **Upcoming meetings** — Meetings, type `calendar`, configure: `CALENDAR BY "Date"`
3. **Pipeline** — Leads, type `board`, configure: `GROUP BY "Stage"`
4. **Active engagements** — Engagements, type `table`, configure: `FILTER "Status" = "Active" SHOW "Client", "Status", "Rate", "Start date"`
5. **Content calendar** — Content Calendar, type `calendar`, configure: `CALENDAR BY "Post date"`
6. **Agent ideas** — Agent Ideas, type `board`, configure: `GROUP BY "Status"`

These are linked views — they point at the real databases, so edits in either place stay in sync. If a `create-view` call fails, note which view and continue; a missing dashboard view is cosmetic, not setup-blocking.

## Step 3 — Collect the rest of the config

Gather the remaining values conversationally and assemble the full config:

- `user.name`, `user.company`
- `voice.name` (their first name), `voice.style_guide_path` (path to their own writing-style file if they have one; otherwise leave it an empty string for a neutral plain-professional voice)
- `email.monitored_addresses` (every address that lands in their connected Gmail), `email.send_as`
- `firewall.no_connector_accounts` (any client whose systems must never be connected; empty list if none)
- `engagements` (empty list at setup; engagement-specific skills fill this later)

## Step 4 — Write the config file

Write `solo-os-config.json` to the project folder confirmed in Step 1 with `_version` `"1.0"`, the captured Notion IDs, the home page ID, and the values from Step 3. Use `config.example.json` in this plugin as the shape. Confirm the file path back to the user.

Never write real IDs or emails into `config.example.json` — that file stays sanitized.

## Step 5 — Verify before handing off

Do not declare success until these pass:

1. For each of the six config IDs, call Notion `fetch` and confirm it resolves and the title matches (Tasks, Meetings, Engagements, Leads & Opportunities, Content Calendar, Agent Ideas).
2. Confirm the four relations resolved with the right names: Tasks has `Engagement` and `Meeting`; Engagements has `Tasks`, `Meetings`, and `Source lead`; Meetings has `Action items`; Leads has `Converted to`.
3. Confirm the home page shows the six dashboard views.
4. Confirm Gmail and Calendar each return data (a recent message, an upcoming event).
5. Run the daily check-in end to end. If it produces a clean today-view with no missing-key errors, onboarding succeeded.

Report each check as pass or fail. If anything fails, name it and stop there — a half-built OS that reports its gap beats a silent one.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Honor `firewall.no_connector_accounts` from the moment it is set.
- Create databases only under the user's chosen home page, never at workspace root unless they ask.
- If `solo-os-config.json` already exists, do not overwrite it silently — show what is there and ask whether to repair specific keys or start fresh.
- Generalize instance-specific labels per `references/notion-schema.md`: the Tasks `Source` "Owner" option and the single Content `Series` "Standalone" option ship neutral, not as the reference user's values.
