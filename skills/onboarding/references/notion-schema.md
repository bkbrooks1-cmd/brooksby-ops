# Notion Schema — Provisioning Spec

The seven databases the Solopreneur OS runs on, captured from the reference workspace. Onboarding recreates these in a new user's Notion, then writes the resulting data-source IDs into `solo-os-config.json`.

This is the source of truth for the schema. If a consuming skill needs a new property, add it here first, then update the skill.

---

## Critical: build order (the relation web)

Several of the seven databases reference each other through two-way relations. A relation property cannot be created until the database it points to exists. So provisioning is a **two-pass** job:

**Pass 1 — create all seven databases with their non-relation properties only.** Capture each new data-source ID.

**Pass 2 — add the relation properties**, now that every target database exists. Notion two-way relations create the paired property on the other side automatically, so create each relation once from the side listed below; do not also create its mirror by hand.

Relations to create in pass 2 (create once, from this side):

| Create on | Property | Points to | Auto-creates mirror |
|---|---|---|---|
| Tasks | Engagement | Engagements | Engagements → Tasks |
| Tasks | Meeting | Meetings | Meetings → Action items |
| Meetings | Engagement | Engagements | Engagements → Meetings |
| Leads & Opportunities | Converted to | Engagements | Engagements → Source lead |
| Content Ideas | Calendar post | Content Calendar | Content Calendar → Idea source |

Agent Ideas is the only database with no relations — it is created complete in pass 1. Content Calendar and Content Ideas have their non-relation properties created in pass 1; the Content Ideas → Content Calendar relation is added in pass 2, once both exist.

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

### The internal engagement buckets

Engagements holds more than clients. **Every task in the OS belongs to an engagement** — a null Engagement is a defect, not a state — so non-client work needs somewhere to live. Four rows in this same table carry it:

| Client (title) | Icon | Config key | Holds |
|---|---|---|---|
| Marketing Content | 📣 | `notion.internal_engagements.marketing_content` | LinkedIn posts, newsletter issues, the Wednesday engine, metrics logging, cadence seeding |
| Business Admin | 🧾 | `notion.internal_engagements.business_admin` | Internal ops: tool and workspace cleanup, system maintenance, taxes, subscriptions |
| Business Development | 🎯 | `notion.internal_engagements.business_development` | Prospect and referral follow-ups with a named opportunity but no engagement yet |
| Networking | 🤝 | `notion.internal_engagements.networking` | Relationship maintenance with no specific opportunity |

These rows carry **only a title and an icon**. Status, Rate, Billing model, and Start date stay empty and `Weekly report` stays unchecked. That emptiness is load-bearing: the `Status = Active` views and the revenue reporting key off those fields, and filling any of them drags an internal bucket into client reporting. Each page body carries a note saying so.

Onboarding creates them in Step 2.4 and writes their page IDs into config. Full routing rules — including that client invoicing follow-through links to the *client* engagement, not Business Admin — are in `references/engagement-routing.md` at the plugin root.

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

Config key: `notion.content_db`. Title property: **Title** (title). This is the post pipeline **and** the published log: a post moves Idea → Drafted → Scheduled → Posted, or Drafted → Archived when a piece is killed rather than deferred, the full draft/final copy lives in the **page body**, and analytics land in the properties once posted.

| Property | Type | Options / config |
|---|---|---|
| Title | title | — |
| Status | select | Idea (gray), Drafted (yellow), Scheduled (orange), Posted (green), Archived (red) |
| Platform | select | LinkedIn (blue), Substack (orange), Both (purple) |
| Series | select | Standalone (gray) — users add their own series options |
| Series week | text | e.g. "Wk 2", "PM-1" |
| Post date | date | — |
| Live URL | url | the live link, set at publish by mark-published |
| Hook | text | the first line / opening hook |
| Hashtags | text | — |
| Impressions | number | analytics (LinkedIn, manual — owner-only) |
| Reactions | number | analytics (LinkedIn, auto) |
| Comments | number | analytics (LinkedIn, auto) |
| Reposts | number | analytics (LinkedIn, auto) |
| Profile views | number | analytics (LinkedIn, manual — owner-only) |
| Opens | number | analytics (Substack, manual — dashboard) |
| Performance notes | text | — |
| Idea source | relation → Content Ideas | pass 2 (mirror of Content Ideas → Calendar post; may be auto-created) |

Note: **Series** ships with only "Standalone"; users add their own series names (the reference workspace uses List of Demands, Outgrown, PM Track). The full post copy is written in the page body, not a property. `Live URL` is set at publish time by the mark-published skill. The analytics columns split by how they're gathered: `Reactions`/`Comments`/`Reposts` are public LinkedIn counts (auto-pullable via Apify); `Impressions`/`Profile views` are owner-only LinkedIn analytics (manual); `Opens` is Substack (manual dashboard). The collect-metrics skill fills these.

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

## Database 7 — Content Ideas

Config key: `notion.content_ideas_db`. Title property: **Idea** (title). A backlog of LinkedIn/Substack article ideas with monetization angles; an idea graduates into a Content Calendar post via the relation.

| Property | Type | Options / config |
|---|---|---|
| Idea | title | — |
| Platform | select | LinkedIn (blue), Substack (orange), Both (purple) |
| Status | select | Backlog (gray), Next (yellow), Drafting (blue), Promoted (green), Dropped (red) |
| Hook | text | the premise / opening line |
| Theme | text | freeform comma-separated tags (e.g. "Discovery, AI, Client story") |
| Topic | text | — |
| Angle | text | the specific hook or thesis |
| Monetization | multi-select | Consulting lead-gen (green), Substack paid subs (orange), Product/course/workshop (purple) |
| Monetization notes | text | — |
| Priority | select | High (green), Medium (yellow), Low (gray) |
| Effort | select | S (green), M (yellow), L (red) |
| Target date | date | — |
| Notes | text | — |
| Calendar post | relation → Content Calendar | pass 2 (auto-creates "Idea source" on Content Calendar) |

Note: the Platform and Monetization options are sensible starting defaults; a new user can adjust them to their own channels and business model.

---

## After provisioning

1. Write each new data-source ID into `solo-os-config.json` under the matching `notion.*` key.
2. Set `notion.home_page` to the parent page the databases were created under.
3. Seed the four internal engagement buckets in Engagements and write their page IDs into `notion.internal_engagements`.
4. Run the verification pass:
   - each of the seven data-source IDs resolves and returns its expected properties;
   - all four `notion.internal_engagements` IDs resolve, titles match, and each has Status, Rate, Billing model, and Start date empty with `Weekly report` unchecked;
   - a query for open Tasks with a null Engagement returns zero rows;
   - a daily check-in runs end to end against the empty workspace.

## Instance-specific values to genericize in the shipped template

These came from the reference workspace and should not ship as-is:

- Tasks → Source → "Brian" option (use the user's name or "Owner").
- Content Calendar → Series → "Outgrown" option (a personal content series).
- Engagements → "Weekly report" checkbox is generic and stays, but its consuming agent (Wednesday weekly report) is a Brian-only engagement skill for now.
