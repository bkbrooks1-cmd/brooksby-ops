# Notion Schema — Provisioning Spec

The six databases the Solopreneur OS runs on, captured from the reference workspace. Onboarding recreates these in a new user's Notion, then writes the resulting data-source IDs into `solo-os-config.json`.

This is the source of truth for the schema. If a consuming skill needs a new property, add it here first, then update the skill.

---

## Critical: build order (the relation web)

Four of the six databases reference each other through two-way relations. A relation property cannot be created until the database it points to exists. So provisioning is a **two-pass** job:

**Pass 1 — create all six databases with their non-relation properties only.** Capture each new data-source ID.

**Pass 2 — add the relation properties**, now that every target database exists. Notion two-way relations create the paired property on the other side automatically, so create each relation once from the side listed below; do not also create its mirror by hand.

Relations to create in pass 2 (create once, from this side):

| Create on | Property | Points to | Auto-creates mirror |
|---|---|---|---|
| Tasks | Engagement | Engagements | Engagements → Tasks |
| Tasks | Meeting | Meetings | Meetings → Action items |
| Meetings | Engagement | Engagements | Engagements → Meetings |
| Leads & Opportunities | Converted to | Engagements | Engagements → Source lead |

Content Calendar and Agent Ideas have no relations — they can be created complete in pass 1.

If the connector's `create-view`/relation support turns out not to set the mirror name (e.g. it defaults to "Related to Tasks"), rename the mirror to match the names above so the skills find their columns.

---

## Database 1 — Tasks

Config key: `notion.tasks_db`. Title property: **Name** (title).

| Property | Type | Options / config |
|---|---|---|
| Name | title | — |
| Status | select | To do (red), In progress (yellow), Waiting (orange), Done (green) |
| Type | select | Deliverable (blue), Prep (purple), Follow-up (yellow), Admin (gray), Networking (red) |
| Priority | select | P1 (red), P2 (yellow), P3 (gray) |
| Source | select | Meeting (blue), Email (purple), Planning (green), Ad hoc (gray), Brian (brown) |
| Due date | date | — |
| Engagement | relation → Engagements | pass 2 |
| Meeting | relation → Meetings | pass 2 |

Note: the **Source → "Brian"** option is voice/instance-specific. In the shipped template, generalize the label (e.g. "Owner" or the user's first name from `voice.name`).

---

## Database 2 — Meetings

Config key: `notion.meetings_db`. Title property: **Name** (title).

| Property | Type | Options / config |
|---|---|---|
| Name | title | — |
| Date | date | — |
| Attendees | text | — |
| Granola link | url | — |
| Engagement | relation → Engagements | pass 2 |
| Action items | relation → Tasks | pass 2 (mirror of Tasks → Meeting; may be auto-created) |

---

## Database 3 — Engagements

Config key: `notion.engagements_db`. Title property: **Client** (title).

| Property | Type | Options / config |
|---|---|---|
| Client | title | — |
| Status | select | Proposal (yellow), Active (green), Paused (orange), Closed (gray) |
| Billing model | select | T&M (blue), Fixed (purple), Retainer (green) |
| Rate | number | format: dollar |
| Start date | date | — |
| Key contacts | text | — |
| Place | place | — |
| Weekly report | checkbox | drives the Wednesday weekly-report agent |
| Tasks | relation → Tasks | pass 2 (mirror; may be auto-created) |
| Meetings | relation → Meetings | pass 2 (mirror; may be auto-created) |
| Source lead | relation → Leads & Opportunities | pass 2 (mirror; may be auto-created) |

---

## Database 4 — Leads & Opportunities

Config key: `notion.leads_db`. Title property: **Name** (title).

| Property | Type | Options / config |
|---|---|---|
| Name | title | — |
| Company | text | — |
| Stage | select | Lead (blue), Qualified (yellow), Proposal (orange), Won (green), Lost (gray) |
| Source | select | LinkedIn (blue), Email inquiry (purple), Referral (green), Conversation (yellow), Calendly (orange) |
| Estimated value | number | format: dollar |
| Next action | text | — |
| Next action date | date | — |
| Notes | text | — |
| Converted to | relation → Engagements | pass 2 |

---

## Database 5 — Content Calendar

Config key: `notion.content_db`. Title property: **Title** (title). No relations.

| Property | Type | Options / config |
|---|---|---|
| Title | title | — |
| Status | select | Idea (gray), Drafted (yellow), Scheduled (orange), Posted (green) |
| Series | select | Outgrown (blue), Standalone (gray) |
| Post date | date | — |
| Performance notes | text | — |

Note: the **"Outgrown"** series option is instance-specific (a Brian content series). Drop or genericize it in the shipped template; keep "Standalone".

---

## Database 6 — Agent Ideas

Config key: `notion.agent_ideas_db`. Title property: **Idea** (title). No relations.

| Property | Type | Options / config |
|---|---|---|
| Idea | title | — |
| Status | select | Backlog (gray), Next (yellow), Built (green), Dropped (red) |
| Value | select | High (green), Medium (yellow), Low (gray) |
| Effort | select | S (green), M (yellow), L (red) |
| Problem it solves | text | — |
| Trigger and data | text | — |

---

## After provisioning

1. Write each new data-source ID into `solo-os-config.json` under the matching `notion.*` key.
2. Set `notion.home_page` to the parent page the databases were created under.
3. Run the verification pass: confirm each of the six IDs resolves and returns its expected properties, then run a daily check-in end to end against the empty workspace.

## Instance-specific values to genericize in the shipped template

These came from the reference workspace and should not ship as-is:

- Tasks → Source → "Brian" option (use the user's name or "Owner").
- Content Calendar → Series → "Outgrown" option (a personal content series).
- Engagements → "Weekly report" checkbox is generic and stays, but its consuming agent (Wednesday weekly report) is a Brian-only engagement skill for now.
