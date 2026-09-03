# Notion schema — rationale and property reference

The databases the Solopreneur OS runs on. **`SKILL.md` Step 2 is the
provisioning procedure and it wins on any disagreement** — it is the
dry-run-verified path, it is self-contained, and reading this file is optional.
What lives here is the per-property detail and the reasoning behind it: what each
column is for, which skill fills it, and which ones cannot be dropped.

If a consuming skill needs a new property, add it here and to `SKILL.md` Step 2
together. A property in one and not the other is how a fresh install ends up
missing a column that some skill writes to silently.

---

## Build order — one ordered pass

Provisioning is **a single ordered pass**, not a create-then-relate two-pass job.
Each two-way relation is declared inline as
`RELATION('<target-ds-id>', DUAL '<mirror name>')`, which auto-creates the
correctly named mirror on the target. A relation can only point at a database
that already exists, so the order below is what makes one pass possible.

| # | Database | Depends on | Config key |
|---|---|---|---|
| 1 | Engagements | — | `notion.engagements_db` |
| 2 | Meetings | Engagements | `notion.meetings_db` |
| 3 | Tasks | Engagements, Meetings | `notion.tasks_db` |
| 4 | Leads & Opportunities | Engagements | `notion.leads_db` |
| 5 | Content Calendar | — | `notion.content_db` |
| 6 | Agent Ideas | — | `notion.agent_ideas_db` |
| 7 | Content Ideas | Content Calendar | `notion.content_ideas_db` |
| 8 | Clips (conditional) | Content Calendar, Content Ideas | `content.notion.clips_db` |

Engagements is created first and declares no outgoing relations — every relation
property on it arrives as a mirror. Agent Ideas has no relations at all. Clips is
created **only when `content.backend` is `"notion"`**; skip it entirely
otherwise, and say which you did.

Never create a mirror by hand. If the connector fails to set a mirror's name and
defaults it to something like "Related to Tasks", rename it to match the name in
the DDL — the skills look that column up by name.

---

## 1 — Engagements

Title: **Client**. The spine of the OS: every task, meeting, and action item
links here, including non-client work.

| Property | Type | Notes |
|---|---|---|
| Client | title | — |
| Status | select | Proposal, Active, Paused, Closed |
| Billing model | select | T&M, Fixed, Retainer |
| Rate | number (dollar) | — |
| Start date | date | — |
| Key contacts | text | — |
| Place | text | see the limitation below |
| Weekly report | checkbox | drives the weekly-report agent |
| Tasks · Meetings · Source lead | relation mirrors | arrive automatically |

The four **internal engagement buckets** are rows in this database, not a
separate concept. They carry no Status, Rate, Billing model, or Start date, and
`Weekly report` stays unchecked — that emptiness is exactly what keeps them out
of revenue reporting and the Status = Active views, including the deliverable
radar. Routing rules live in `references/engagement-routing.md`.

Known limitation: Notion's `place` property type cannot be created through DDL,
so `Place` is provisioned as text. Converting it in the Notion UI afterward is
optional and never setup-blocking.

## 2 — Meetings

Title: **Name**. One page per captured meeting, minutes in the page body.

| Property | Type | Notes |
|---|---|---|
| Name | title | — |
| Date | date | — |
| Attendees | text | — |
| Granola link | url | **the dedupe key**, and the path to the full transcript |
| Engagement | relation → Engagements | mirror: `Meetings` |
| Action items | relation mirror | from Tasks |

`Granola link` carries the meeting id and is what `capture` step 2 dedupes on. A
failed dedupe re-summarizes the meeting from scratch, so the duplicate usually
carries a *different title for the same call* — which is why the audit query
groups on this column and never on name or date.

## 3 — Tasks

Title: **Name**. Everything the OS asks anyone to do.

| Property | Type | Notes |
|---|---|---|
| Name | title | phrased as an outcome |
| Status | select | To do, In progress, Waiting, Done |
| Type | select | Deliverable, Prep, Follow-up, Admin, Networking |
| Priority | select | P1, P2, P3 |
| Source | select | Meeting, Email, Planning, Ad hoc, Owner |
| Due date | date | — |
| Engagement | relation → Engagements | mirror: `Tasks`. **Never null** |
| Meeting | relation → Meetings | mirror: `Action items` |

`Source` ships with **Owner** as the self-generated option. Rename it to the
user's own preference if they want; do not ship anyone's first name as an option
value.

## 4 — Leads & Opportunities

Title: **Name**.

| Property | Type | Notes |
|---|---|---|
| Name | title | — |
| Company | text | — |
| Stage | select | Lead, Qualified, Proposal, Won, Lost |
| Source | select | LinkedIn, Email inquiry, Referral, Conversation, Calendly |
| Estimated value | number (dollar) | — |
| Next action | text | — |
| Next action date | date | friday-wrap flags these when past due |
| Notes | text | — |
| Converted to | relation → Engagements | mirror: `Source lead` |

## 5 — Content Calendar

Title: **Title**. The post pipeline **and** the published log at once: a piece
moves Idea → Drafted → Scheduled → Posted, or Drafted → Archived when it is
killed rather than deferred. The full draft or final copy lives in the **page
body**, never in a property.

| Property | Type | Filled by |
|---|---|---|
| Title | title | — |
| Status | select | Idea, Drafted, Scheduled, Posted, Archived |
| Platform | select | LinkedIn, Substack, Both |
| Series | select | ships with Standalone only; the user adds their own |
| Series week | text | e.g. "Wk 2" |
| Post date | date | mark-published |
| Live URL | url | mark-published, at publish time |
| Hook | text | the first line of the published piece |
| Hashtags | text | — |
| Impressions | number | collect-metrics — owner-only, manual |
| Reactions | number | collect-metrics — public, auto |
| Comments | number | collect-metrics — public, auto |
| Reposts | number | collect-metrics — public, auto |
| Profile views | number | collect-metrics — owner-only, manual |
| Opens | number | collect-metrics — Substack, manual |
| Performance notes | text | collect-metrics |
| Idea source | relation mirror | from Content Ideas |
| Source clips | relation mirror | from Clips, on the notion backend only |

**The seven analytics-and-link columns are required, not optional.** `Live URL`,
`Comments`, `Reposts`, `Profile views`, and `Opens` were missing from earlier
provisioning, and the failure is silent: the writing skill has nowhere to put the
number, writes what it can, and reports success. `Archived` is likewise a real
status, not decoration — without it a killed piece has to be deleted or left
looking scheduled.

The analytics split by how they are gathered, which is why they are listed that
way above: three are public counts a scraper can read, three are owner-only, and
an empty one means *pending*, never zero.

Whatever `Series` options the user ends up with go into `content.series` in
config. Skills read series names from there and never from their own body.

## 6 — Agent Ideas

Title: **Idea**. No relations; created complete.

| Property | Type | Notes |
|---|---|---|
| Idea | title | — |
| Status | select | Backlog, Next, Built, Dropped |
| Value | select | High, Medium, Low |
| Effort | select | S, M, L |
| Problem it solves | text | — |
| Trigger and data | text | — |

This database is deliberately outside the content system. An `agent` idea
captured from email lands here and never enters the content pipeline.

## 7 — Content Ideas

Title: **Idea**. The backlog of angles; an idea graduates into a Content Calendar
post through the relation.

| Property | Type | Notes |
|---|---|---|
| Idea | title | — |
| Platform | select | LinkedIn, Substack, Both |
| Status | select | Backlog, Next, Drafting, Promoted, Dropped |
| Hook | text | the premise / opening line |
| Theme | text | freeform comma-separated tags |
| Topic | text | — |
| Angle | text | the specific thesis |
| Monetization | multi-select | Consulting lead-gen, Substack paid subs, Product/course/workshop |
| Monetization notes | text | — |
| Priority | select | High, Medium, Low |
| Effort | select | S, M, L |
| Target date | date | — |
| Notes | text | — |
| Calendar post | relation → Content Calendar | mirror: `Idea source` |
| Source clips | relation → Clips | added with Clips, on the notion backend only |

Platform and Monetization options are sensible starting defaults; a new user
adjusts them to their own channels and business model.

## 8 — Clips (conditional)

Title: **Name**. Created **only when `content.backend` is `"notion"`**. Source
material kept from elsewhere; the full clip text lives in the page body.

| Property | Type | Notes |
|---|---|---|
| Name | title | — |
| URL | url | **the dedupe key**: one URL is one clip |
| Source | select | Substack, LinkedIn, Web |
| Why | text | the one-line reason it was kept |
| Captured | date | — |
| Used in | relation → Content Calendar | mirror: `Source clips` |
| Feeds ideas | relation → Content Ideas | mirror: `Source clips` |

Both relations are required. Together they make provenance bidirectional: with
`Used in` alone a clip can say what it became but not what idea it fed, and the
"did this clip ever earn its keep" question stops being answerable as a query.
The weekly hygiene check keys off exactly these two columns.

`Source` carries the one piece of information a folder-based store encoded in its
structure rather than its content.

The authority for this schema, the field map, and the per-backend procedures is
`references/content-storage.md`. If that file and this one disagree, that one
wins.

---

## After provisioning

1. Write each data-source ID into `solo-os-config.json` under its config key from
   the build-order table.
2. Set `notion.home_page` to the parent page everything was created under, and
   `notion.week_plans_page` to the Weekly Plans and wraps page.
3. Seed the four internal engagement buckets and write their page IDs into
   `notion.internal_engagements`.
4. On the notion backend, create the Theme Map page and write its ID into
   `content.notion.theme_map_page`.
5. Run the verification pass in `SKILL.md` Step 5. In short: every ID resolves
   with the expected title and properties; every relation resolved with the right
   mirror name; the four buckets resolve with Status, Rate, Billing model, and
   Start date empty and `Weekly report` unchecked; all three queries in
   `references/engagement-routing.md` return zero rows; a daily check-in runs end
   to end.

## Nothing instance-specific ships here

Every option value in this file is either functional or a neutral default. Two
that have to stay that way:

- **Tasks → Source** ships `Owner`, not anyone's first name.
- **Content Calendar → Series** ships `Standalone` alone. Series names are user
  content and belong in `content.series`, never in a shipped option list or in a
  skill body.

`Weekly report` on Engagements is generic and stays. Its consuming
weekly-report agent is an engagement-specific skill that does not ship in this
plugin.
