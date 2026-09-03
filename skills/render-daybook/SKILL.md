---
name: render-daybook
description: Render the Daybook — one page showing what is true right now across client work, the schedule, and money. Use when the user says "refresh my daybook", "show my daybook", "build my daybook", "update the daybook", or asks to see the dashboard. Also runs as the last phase of daily-checkin. Reads Notion, Google Calendar and (if configured) QuickBooks, and publishes a Cowork artifact. Reads only; never writes to Notion.
---

# Render daybook

One page that answers "what is true right now." It is a read layer over the databases the rest of the OS already writes to. Every number on it traces back to a row somebody or something already created.

The artifact is a snapshot, not a live view. It says so on its face.

Runs standalone on request, and as phase 3 of `daily-checkin` — always last, after that skill's writes have been confirmed, so the snapshot reflects them.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before rendering the Daybook."

Required keys: `notion.tasks_db`, `notion.engagements_db`, `notion.meetings_db`, `daybook.timezone`, `daybook.stale_hours`, `daybook.work_window_days`, `daybook.tiles`.

Used if present: `daybook.artifact_id`, `daybook.connectors`, `daybook.optional_connectors`, `daybook.revenue`, `firewall.no_connector_accounts`, `notion.internal_engagements`.

`notion.internal_engagements` names the four non-client buckets so the work-by-engagement tile can order them below client groups. If it is absent, order every group by due date alone and render normally — a missing key degrades the sort, it does not stop the Daybook.

**`daybook.artifact_id` is what keeps the link stable.** Read it before rendering. If it is empty, call `create_artifact` and write the id back into config. If it is set, call `update_artifact` against it. Never create a second artifact — a new id strands the user's pinned entry in the gallery.

## The QuickBooks rule

`daybook.revenue.source` accepts `"quickbooks"` or `"none"`. When it is `"none"`, or `daybook.tiles.revenue` is false, the revenue tile does not render and no QuickBooks call is made. The shipping template `config.example.json` sets `"none"` with `tiles.revenue: false`, so a fresh install has no QuickBooks dependency at all.

The same rule applies to any connector added for a single instance: default off, config-gated, absent from the example config.

## Sources to read

One query per Notion data source, maximum. **Notion is on the Business plan — querying across multiple data sources returns `entitlement_required` and is unavailable.** Every join happens in the render logic, not in Notion. This is a standing constraint for the whole OS.

1. **Connector health** — run the probes in `${CLAUDE_PLUGIN_ROOT}/references/connector-health.md` before anything else. Their results drive the health band and decide which tiles degrade.
2. **Notion Tasks** — `collection://{notion.tasks_db}`, one query, `Status != 'Done'` or `Status IS NULL`. Select only the columns needed: id, url, Name, Type, Priority, Status, Source, Engagement, Meeting, `date:Due date:start`, createdTime. Relation columns return JSON arrays of page URLs, not names — resolve them against the Engagements and Meetings results.
3. **Notion Engagements** — `collection://{notion.engagements_db}`, one query. Select id, url, Client, Status, Billing model, Rate. Do not `SELECT *`: the Tasks and Meetings relation columns on this table are hundreds of URLs long and will blow the response budget.
4. **Notion Meetings** — `collection://{notion.meetings_db}`, one query, most recent rows. Used to resolve the Meeting relation on tasks.
5. **Google Calendar** — today and tomorrow, all calendars, ordered by start time, in `daybook.timezone`.
6. **QuickBooks** — only when the revenue tile is on. Invoices for the current month, A/R aging summary. Nothing else.

## Bands

Skip any band whose switch in `daybook.tiles` is false.

### Band 1 — status strip

| Tile | Number | Subtext |
|---|---|---|
| Today | count of timed calendar events today | next event, time and title |
| Due today | tasks due today | overdue count · total open |
| Waiting | tasks at Status = Waiting | oldest item and how long it has sat, by createdTime |
| Revenue | invoiced month to date | open AR · overdue AR · progress to target |

Overdue means a due date strictly before today on a task that is not Done. Total open is every task the Tasks query returned.

Revenue shows three numbers because a consulting practice has no MRR. Invoiced MTD is the achievement, open AR is the float, overdue AR is the problem. `revenue.target_basis` decides what the progress bar fills against: `invoiced` counts what went out the door this month, `collected` counts what landed. Set `monthly_target` to 0 and the bar hides, leaving the bare figures.

Never print a client name on the revenue tile. The tile is three numbers.

### Band 2 — two columns

**Left: work by engagement.** Tasks due from today through `daybook.work_window_days` ahead, grouped by the Engagement relation. Each group shows the engagement name, its count, and up to three task lines with priority chip and due date. Overflow collapses to "+N more".

Order in two tiers, client groups above internal buckets:

1. **Client groups** — any engagement not listed in `notion.internal_engagements` — ordered by soonest due date.
2. **Internal buckets** — the four rows in `notion.internal_engagements` (Marketing Content, Business Admin, Business Development, Networking) — ordered by soonest due date among themselves.

The tiering is the point. Sorting on due date alone lets a Marketing Content group outrank a client with work due the same day, which reads as the wrong priority at a glance. Mark internal buckets visually — a muted label or a subtler chip — so the split is readable without counting rows.

An **Unassigned** group catches tasks with no engagement, and renders last. Under the routing convention a null Engagement is a defect, not a state, so this group appearing at all means a skill wrote a null. Never hide it, and label it so it reads as a problem rather than a category.

**Right, stacked:**

- **Today's schedule** — calendar events with time, title, and location or meeting link. Each event gets a prep chip: green when a Prep task for it is Done, amber when one is open, gray when none exists. Tomorrow's meetings that need prep today go below a divider.

  Match a prep task to an event by the `Meeting` relation first. Prep tasks are usually created before the meeting page exists, so the relation is often empty — fall back to the naming convention `build-prep` writes: `Prep: <meeting name> (<Day M/D H:MM>)`. Match the event summary inside the task name.

- **Connector health** — renders only when something is wrong. See `${CLAUDE_PLUGIN_ROOT}/references/connector-health.md` for the band's contents.

### Band 3 — deliverable radar

One row per engagement at Status = Active: engagement, next deliverable, status, due date.

The internal engagement buckets carry no Status, so this filter excludes them without needing a rule about them. **Verify that after any change to the buckets rather than assuming it** — an internal bucket showing up in the radar is the first sign the Status filter is not doing what this doc says, and the tile drops rows silently. The next deliverable is the soonest-due open task of Type = Deliverable for that engagement. Break ties on due date by priority, then by earliest createdTime, so two renders in a row show the same row.

An engagement at Status = Active with no dated deliverable renders as "no dated deliverable" in muted text rather than being dropped. A client with nothing scheduled is a fact worth seeing.

The Tasks schema has no owner property, so the radar has no Owner column. Add one only after Tasks gains the field.

### Footer

Data-as-of timestamp, the connector status row, and deep links into the Notion Tasks, Engagements and Meetings databases.

## Staleness

Stamp a UTC timestamp into every render and show its age in the header in plain language. Past `daybook.stale_hours` (default 26, so a missed weekday shows up but a weekend does not), the header turns amber and reads "last refreshed <when> — say refresh my daybook".

Write this for someone who has never read a guide. The artifact looks live and is not; the header is the only thing that says so.

## Render limits

Artifact HTML streams token by token. A large render flickers and truncates.

- Cap each group at three rows and collapse the rest to "+N more".
- Keep the whole document under roughly 1,200 lines.
- Inline all CSS. No external fonts — name Inter and DM Sans first in the stack and let the system fall back.
- Body text is 18px. Nothing on the page goes below 13.5px, including chips and column headers. This is a wall-glanceable page, not a dense report; if a row does not fit at 18px, cut the row rather than the type size.
- Dark: set `color-scheme: dark`. Background is deep navy, cards one step lighter so they lift off it, and tiles one step lighter again. Teal carries engagement names, times and the progress bar; red and amber are reserved for failure and staleness so they still mean something. Brand palette is Navy #0D2240 and Teal #1A9E7A, darkened and brightened respectively for a dark surface.

## Hard rules

- **This skill reads. It never writes to Notion.** Every write in the OS goes through a skill with a confirmation step. A broken Daybook must not be able to corrupt data.
- The one write it makes anywhere is `daybook.artifact_id` back into `solo-os-config.json`, once, after the first successful create.
- A failed source means a smaller Daybook, not a silent one. A tile whose connector is down renders its last known value with a muted "as of" note; it never blanks and it never shows a zero it did not measure.
- Never invent a number. If a source returned nothing, say the source returned nothing.
- Accounts in `firewall.no_connector_accounts` are never connected. The Daybook touches no content storage at all — it reads Tasks, Engagements, Meetings, and the calendar, and the only thing it ever writes is `daybook.artifact_id` — so no client material can reach the content system through this skill on either backend. An engagement listed in `firewall.walled_engagements` still never appears on a tile.
- `solo-os-config.json` is live and private. Never print it, paste it into a deliverable, or copy values out of it into documentation.
