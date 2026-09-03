---
name: onboarding
description: Stand up the Solopreneur OS for a new user from an empty Notion workspace. Use when someone says "set up the OS", "onboard me", "first-time setup", "get me started", "install the Solopreneur OS", or when any other skill reports that solo-os-config.json is missing. Provisions the Notion databases, captures their IDs into config, connects Gmail/Calendar/Notion, and verifies the install with a check-in.
---

# Onboarding

Take a new user from nothing to a working daily check-in. This skill builds the Notion databases the OS runs on, writes their IDs into `solo-os-config.json`, walks through connecting the tools, and verifies the whole thing before handing off.

Run it once per install. It is the make-or-break setup flow: if it works unattended, the OS travels to a new person; if it doesn't, nothing else matters.

## Before you start

Confirm the prerequisites in plain language. The user needs:

1. The **Notion** connector enabled in Cowork.
2. The **Gmail** and **Google Calendar** connectors enabled (the daily skills read both).
3. One Notion page they will let the OS use as its home — new or existing — shared with the Notion integration so it can create databases under it (in Notion: open the page → ••• menu → Connections → add the integration).
4. A Cowork project folder for the OS, where `solo-os-config.json` will be written.

**Granola** is optional but needed for meeting capture (used by the check-in and the planner to pull recent meetings into Notion minutes and action items). Connect it too if the user wants that; the OS works without it, just without automatic meeting capture.

If a connector is missing, stop and tell the user exactly which one to enable in Settings, then resume. Do not proceed past a missing Notion connector — every step below depends on it.

## Step 1 — Identify the workspace and home page

Call Notion `fetch` with id `"self"` to confirm the connected workspace and the user's identity. State the workspace name back to the user so they know where the databases will land.

Ask the user for the **home page**: the parent under which the databases will be created. Accept a Notion URL or let them name a page to search for. Resolve it to a page ID. This ID becomes `notion.home_page` in config. If they have no page yet, create one titled with their company or "Solopreneur OS" and use that.

Then confirm the **project folder**: ask the user which Cowork project folder the OS should live in. That folder is where `solo-os-config.json` will be written in Step 4. Do not assume a folder — confirm it explicitly, and create it if it does not exist.

## Step 1.5 — Verify Notion access before building

Before creating anything real, confirm the integration can write under the home page. Create one throwaway database under it (a single-column "Access check") and confirm it returns an ID, then trash it. If creation is denied, stop and tell the user in plain language: open the home page in Notion, click the ••• menu, choose Connections, add the integration, then resume. This catches the most common setup failure — the integration not being granted page access — before any real work begins.

## Step 2 — Provision the databases

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

**5. Content Calendar** (the post pipeline + published log; the draft/final copy lives in each page's body):
```
CREATE TABLE ("Title" TITLE, "Status" SELECT('Idea':gray, 'Drafted':yellow, 'Scheduled':orange, 'Posted':green, 'Archived':red), "Platform" SELECT('LinkedIn':blue, 'Substack':orange, 'Both':purple), "Series" SELECT('Standalone':gray), "Series week" RICH_TEXT, "Post date" DATE, "Live URL" URL, "Hook" RICH_TEXT, "Hashtags" RICH_TEXT, "Impressions" NUMBER, "Reactions" NUMBER, "Comments" NUMBER, "Reposts" NUMBER, "Profile views" NUMBER, "Opens" NUMBER, "Performance notes" RICH_TEXT)
```
Capture as `CONTENT_ID` → config `notion.content_db`. The user adds their own `Series` options, and writes the list they end up with into `content.series` — skills read series names from config, never from their own body.

The analytics columns split by how they are gathered, and `collect-metrics` fills all of them: `Reactions`/`Comments`/`Reposts` are public LinkedIn counts, `Impressions`/`Profile views` are owner-only LinkedIn analytics, `Opens` is Substack. `Live URL` is set at publish time by `mark-published`. **All seven are required, not optional** — a Calendar missing any of them fails the content cycle silently, because the writing skill has nowhere to put the number and reports success anyway. `Archived` is the status for a piece that was killed rather than deferred.

**6. Agent Ideas** (standalone):
```
CREATE TABLE ("Idea" TITLE, "Status" SELECT('Backlog':gray, 'Next':yellow, 'Built':green, 'Dropped':red), "Value" SELECT('High':green, 'Medium':yellow, 'Low':gray), "Effort" SELECT('S':green, 'M':yellow, 'L':red), "Problem it solves" RICH_TEXT, "Trigger and data" RICH_TEXT)
```
Capture as `IDEAS_ID` → config `notion.agent_ideas_db`.

**7. Content Ideas** (relates to Content Calendar — created after it):
```
CREATE TABLE ("Idea" TITLE, "Platform" SELECT('LinkedIn':blue, 'Substack':orange, 'Both':purple), "Status" SELECT('Backlog':gray, 'Next':yellow, 'Drafting':blue, 'Promoted':green, 'Dropped':red), "Hook" RICH_TEXT, "Theme" RICH_TEXT, "Topic" RICH_TEXT, "Angle" RICH_TEXT, "Monetization" MULTI_SELECT('Consulting lead-gen':green, 'Substack paid subs':orange, 'Product/course/workshop':purple), "Monetization notes" RICH_TEXT, "Priority" SELECT('High':green, 'Medium':yellow, 'Low':gray), "Effort" SELECT('S':green, 'M':yellow, 'L':red), "Target date" DATE, "Notes" RICH_TEXT, "Calendar post" RELATION('CONTENT_ID', DUAL 'Idea source'))
```
Capture as `CONTENT_IDEAS_ID` → config `notion.content_ideas_db`. Auto-creates `Idea source` on Content Calendar.

**8. Clips** — **only when `content.backend` is `"notion"`.** Skip it entirely on the `vault` backend and on an install with no `content` block; say which you did. It relates to both content databases, so it is created last, after `CONTENT_ID` and `CONTENT_IDEAS_ID` exist:
```
CREATE TABLE ("Name" TITLE, "URL" URL, "Source" SELECT('Substack':orange, 'LinkedIn':blue, 'Web':gray), "Why" RICH_TEXT, "Captured" DATE, "Used in" RELATION('CONTENT_ID', DUAL 'Source clips'), "Feeds ideas" RELATION('CONTENT_IDEAS_ID', DUAL 'Source clips'))
```
Capture as `CLIPS_ID` → config `content.notion.clips_db`. Auto-creates `Source clips` on Content Calendar and on Content Ideas.

The full clip text lives in the page body, not in a property. `URL` is the dedupe key: one URL is one clip. The two relations are what keep provenance bidirectional — without `Feeds ideas` a clip can say what it became but not what idea it fed. The schema and the field map are in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`; that file is the authority if the two ever disagree.

On the `notion` backend, also create a **Theme Map** page under the home page — one page holding the forward content roadmap — and capture its ID as `content.notion.theme_map_page`.

Known limitation: Notion's `place` property type cannot be created via DDL, so Engagements `Place` is provisioned as text. If the user wants a true location field, they convert it in the Notion UI after setup. Note this and move on — it is not setup-blocking.

## Step 2.4 — Seed the four internal engagement buckets

Every task in the OS belongs to an engagement. Client work links to a client row; everything else links to one of four internal buckets. Seed them now, or the user creates orphan tasks on day one. The full convention is at `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md`.

Create four pages in the Engagements data source (`ENG_ID`), one per bucket:

| Client (title) | Icon | Config key |
|---|---|---|
| Marketing Content | 📣 | `notion.internal_engagements.marketing_content` |
| Business Admin | 🧾 | `notion.internal_engagements.business_admin` |
| Business Development | 🎯 | `notion.internal_engagements.business_development` |
| Networking | 🤝 | `notion.internal_engagements.networking` |

Set the title and the icon. **Leave Status, Rate, Billing model, and Start date empty, and leave `Weekly report` unchecked.** Those four fields are what the `Status = Active` views and the revenue reporting key off; filling any of them pulls an internal bucket into client reporting.

Write this note into each page's body so a later skill — or a later you — does not helpfully fill them in:

> Internal engagement bucket, not a client. Status, Rate, Billing model, and Start date are intentionally empty: those fields drive the Status = Active views and revenue reporting, and an internal bucket must stay out of both. Do not fill them in.

Capture each returned page ID and write it into `notion.internal_engagements` in Step 4.

## Step 2.5 — Build the home dashboard

Give the user a working cockpit on the home page instead of bare databases. Use Notion `create-view` with `parent_page_id` = the home page ID and `data_source_id` = the relevant captured ID. Each call appends a linked view block to the home page. Create these, in order:

1. **Open tasks** — `data_source_id` = Tasks, type `table`, configure: `FILTER "Status" != "Done" SORT BY "Due date" ASC SHOW "Name", "Status", "Priority", "Due date", "Engagement"`
2. **Upcoming meetings** — Meetings, type `calendar`, configure: `CALENDAR BY "Date"`
3. **Pipeline** — Leads, type `board`, configure: `GROUP BY "Stage"`
4. **Active engagements** — Engagements, type `table`, configure: `FILTER "Status" = "Active" SHOW "Client", "Status", "Rate", "Start date"`
5. **Content calendar** — Content Calendar, type `calendar`, configure: `CALENDAR BY "Post date"`
6. **Agent ideas** — Agent Ideas, type `board`, configure: `GROUP BY "Status"`
7. **Content ideas** — Content Ideas, type `board`, configure: `GROUP BY "Status"`
8. **Clips** — only when Clips was provisioned. Clips, type `table`, configure: `SORT BY "Captured" DESC SHOW "Name", "Source", "Why", "Captured", "Used in", "Feeds ideas"`

These are linked views — they point at the real databases, so edits in either place stay in sync. If a `create-view` call fails, note which view and continue; a missing dashboard view is cosmetic, not setup-blocking.

## Step 2.6 — Create the Weekly Plans and wraps page

`monday-planner` and `friday-wrap` both write their output as sub-pages of one shared parent, so a week's plan and its wrap sit together in date order. Without that parent they fall back to the home page, and within a month the home page is a stack of loose week pages.

Create one page under the home page titled **Weekly Plans and wraps** (icon 🗓️). Leave the body empty — the planner and the wrap fill it. Capture the returned page ID and write it into `notion.week_plans_page` in Step 4.

## Step 3 — Collect the rest of the config

Gather the remaining values conversationally and assemble the full config:

- `user.name`, `user.company`
- `voice.name` (their first name), `voice.style_guide_path` (path to their own writing-style file if they have one; otherwise leave it an empty string for a neutral plain-professional voice)
- **`daybook.timezone`** — ask for it in words ("what time zone are you in?") and record the IANA name, e.g. `America/New_York`. **Ask; never infer it** from the workspace, the calendar, or the machine clock. Every "today" in the OS is computed in this zone, so a wrong value silently shifts the whole day view. The rest of the `daybook` block ships at its template defaults and is not asked about.
- `email.monitored_addresses` (every address that lands in their connected Gmail), `email.send_as`
- `firewall.no_connector_accounts` (any client whose systems must never be connected; empty list if none), `firewall.walled_engagements` (any engagement whose name must never enter the content system; empty list if none)
- `capture.lookback` (how far back the capture routine pulls meetings: `this_week`, `last_week`, or `last_30_days`; default `last_week`)
- **`content.backend`** — only if the user wants a content pipeline. `"notion"` unless they specifically want a local markdown store, in which case `"vault"` plus the path keys in `config.example.json`. This is the key that decided whether Step 2 created the Clips database, so it is settled before Step 2 runs, not here — confirm it rather than asking again.
- `content.series` — the series names the user ended up with in the Content Calendar `Series` property. `["Standalone"]` if they added none.
- `voice.register` — `"linkedin"`, `"newsletter"`, or `"none"`. The default register for a user who publishes to only one channel; `"none"` when they publish to both or neither, which is the safe default. `polish-and-score` uses it only to pick a chain when the asset type is genuinely ambiguous; it never overrides an explicit `target`.
- `engagements` (empty list at setup; engagement-specific skills fill this later)

## Step 4 — Write the config file

Write `solo-os-config.json` to the project folder confirmed in Step 1 with `_version` `"1.3"`, the captured Notion IDs, the home page ID, the Weekly Plans and wraps page ID from Step 2.6 under `notion.week_plans_page`, the four internal-bucket page IDs from Step 2.4 under `notion.internal_engagements`, and the values from Step 3. Use `config.example.json` in this plugin as the shape. Confirm the file path back to the user.

**Write the whole `daybook` block**, copied from `config.example.json` with `timezone` set to the answer from Step 3 and `artifact_id` left empty. Do not write a partial block: `render-daybook` requires `timezone`, `stale_hours`, `work_window_days`, and `tiles`, and it stops on the first one missing. Copy `connectors` and `optional_connectors` as they ship, then remove any connector the user did not enable — a name listed there with no tools available is treated as a dead connector, not as "not configured", and will report a failure every morning.

**Path keys are project-folder-relative.** `voice.sources`, `voice.style_guide_path`, and `voice.newsletter_voice_path` are written relative to the project folder from Step 1 — `About Me/voice.md`, never an absolute path. The config travels with the folder, so an absolute path breaks the moment the folder moves or the OS is installed for someone else.

Never write real IDs or emails into `config.example.json` — that file stays sanitized.

## Step 5 — Verify before handing off

Do not declare success until these pass:

1. For each config ID written in Step 4, call Notion `fetch` and confirm it resolves and the title matches (Tasks, Meetings, Engagements, Leads & Opportunities, Content Calendar, Agent Ideas, Content Ideas — plus Clips and the Theme Map page when `content.backend` is `"notion"`).
2. Confirm the relations resolved with the right names: Tasks has `Engagement` and `Meeting`; Engagements has `Tasks`, `Meetings`, and `Source lead`; Meetings has `Action items`; Leads has `Converted to`; Content Ideas has `Calendar post` and Content Calendar has `Idea source`.
3. Confirm the home page shows the dashboard views for every database provisioned, and that `notion.week_plans_page` resolves to a page titled "Weekly Plans and wraps". If that key is missing, `monday-planner` and `friday-wrap` still run — they fall back to the home page and say so — but the fallback is a degraded install, not a passing one.
4. Confirm all four IDs in `notion.internal_engagements` resolve, their titles match (Marketing Content, Business Admin, Business Development, Networking), and each has Status, Rate, Billing model, and Start date empty with `Weekly report` unchecked. A bucket carrying a Status is a failure — it will show up in client reporting.
5. Run all **three** verification queries in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md` — open Tasks with a null Engagement, Meeting pages with a null Engagement, and duplicate meeting pages grouped by `Granola link`. Expect zero rows from each. Each one passes while the others fail, which is why all three run. On a fresh install they pass trivially; run them anyway, because this is the set the user re-runs later and it should be established as normal from day one.
6. Confirm Gmail and Calendar each return data (a recent message, an upcoming event).
7. Run the daily check-in end to end. If it produces a clean today-view with no missing-key errors, onboarding succeeded.

Report each check as pass or fail. If anything fails, name it and stop there — a half-built OS that reports its gap beats a silent one.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Honor `firewall.no_connector_accounts` from the moment it is set.
- Create databases only under the user's chosen home page, never at workspace root unless they ask.
- If `solo-os-config.json` already exists, do not overwrite it silently — show what is there and ask whether to repair specific keys or start fresh.
- Generalize instance-specific labels per `references/notion-schema.md`: the Tasks `Source` "Owner" option and the single Content `Series` "Standalone" option ship neutral, not as the reference user's values.
