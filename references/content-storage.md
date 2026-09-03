# Content storage contract

Shared rule for every skill that reads or writes content — clips, idea notes,
drafts, the logs. Referenced from `capture`, `wednesday-synthesis`,
`draft-content`, `polish-and-score`, `mark-published`, `collect-metrics`,
`friday-wrap`, `monday-planner`, `render-daybook`, and `surface-humanizer`.
Change it here, not in the skills.

---

## The switch

> **Skills never branch on the backend themselves.** They call the six named
> operations below. This file says what each operation does per backend, and
> `content.backend` says which set of procedures is live. Ten skills, one
> switch, zero half-branches.

`content.backend` takes `"vault"` (markdown files on disk) or `"notion"`
(databases through the Notion connector). One backend is live at a time. A skill
that reads one store and writes the other corrupts both, which is the failure
this contract exists to prevent — if a skill touches content storage without
going through a named operation, stop and fix the contract first.

Everything outside the content pipeline is unaffected by the switch. Tasks,
Meetings, Engagements, Leads, and Agent Ideas are Notion on both settings; they
were never storage-coupled.

## The firewall

> **An engagement listed in `firewall.walled_engagements` never enters the
> content system.** Not in a clip, an idea, a draft, a log row, a calendar row,
> a page body, or a provenance field — and not in the text of a rule describing
> the wall. Documenting the exception is not an exception.

The wall is a property of the content system, not of any one backend. It held on
files and it holds on databases; moving stores does not move the line. Skills
read the list from config and never name a walled engagement in their own body.

Two checks that belong to the operations rather than to any one skill:

- **On write.** Before a note, row, or batch is written, check it against the
  configured names. A hit drops the item and surfaces it for the user's call; it
  is never silently sanitized and written anyway.
- **On bulk read.** A sweep of existing material is where this goes wrong
  quietly — most likely in a clip's `Why` line or an idea's provenance note.
  Check the batch, not just the new text.

Accounts in `firewall.no_connector_accounts` never feed content either. An idea
captured from one is generalized and routed to Agent Ideas, or dropped.

---

## The six operations

| Operation | Called by | Reads / writes |
|---|---|---|
| `read a clip` | wednesday-synthesis, capture, draft-content | source material kept from elsewhere |
| `write a note` | capture, draft-content, wednesday-synthesis | a new clip, idea, or draft |
| `read the metrics log` | wednesday-synthesis, collect-metrics, friday-wrap | what published pieces actually did |
| `append to a log` | mark-published, collect-metrics | the published log, the metrics log, the backlog |
| `list drafts` | friday-wrap, monday-planner, polish-and-score, draft-content | unpublished work in progress |
| `update front matter` | mark-published, collect-metrics, capture, wednesday-synthesis | status and provenance on an existing item |

Each has one `vault` procedure and one `notion` procedure. Nothing else touches
storage.

---

### 1. Read a clip

Returns title, URL, the `Why` line, capture date, source channel, full text, and
both halves of provenance. Selectors: by URL (the dedupe path), by title, or
every clip captured or modified inside a window — `content.synthesis_lookback_days`,
default 14.

**vault.** Read markdown under `{content.vault_root}/01-Clips`, including the
`Substack/` and `LinkedIn/` subfolders. Skip `_index.md` and `assets/`.

| Field | From |
|---|---|
| Title | `title` front matter, else the H1 |
| URL | `source` |
| Captured | `clipped`, else `created` |
| Source channel | the subfolder name; `Web` when the clip sits at the folder root |
| Why | the first body line containing `Why:` — older clips carry a personal prefix before it, so match the token, not the line start |
| Text | everything after the front-matter block |
| Used in | the `used-in` list |
| Feeds ideas | the reverse lookup — notes whose `sources` name this file |

Window filter on `clipped`/`created`, falling back to file modified time when
neither is present.

**notion.** Query `collection://{content.notion.clips_db}`:

```sql
SELECT Name, URL, Source, Why, Captured, "Used in", "Feeds ideas"
FROM "collection://{content.notion.clips_db}"
WHERE Captured >= <window start>
```

**The full text is in the page body, not in a property.** A row read alone
returns metadata and no clip. Fetch the page whenever the text is needed —
synthesis scanning `Why` lines can stay on rows; drafting cannot.

Dedupe on `URL` on both backends. One URL is one clip, across the batch and
against everything already stored.

### 2. Write a note

Three kinds: **clip**, **idea**, **draft**. Anything headed for
`capture.idea_capture.personal_ref_path` or the inbox is deliberately outside
this operation — that shelf is not part of the content system, carries no
contract, and never becomes a source for content.

Both backends: propose first, write on one confirmation, and stay idempotent on
the dedupe key — URL for a clip, title for an idea or a draft.

**vault.** Destination by kind:

| Kind | Lands in |
|---|---|
| clip | `{content.vault_root}/01-Clips` + the channel subfolder |
| idea | `content.ideas_path`/`Newsletter-Topics` or `/LinkedIn-Posts` |
| draft (newsletter) | `content.newsletter_drafts_path`, as `Issue 0N - <Title>.md` |
| draft (LinkedIn) | `content.linkedin_drafts_path`, as `Post - <Title>.md` |

Every note is born complete:

- front matter: `type` (clip / idea / draft / session / reference / hub),
  `created` (YYYY-MM-DD), `status` (inbox / active / drafted / published /
  archived), `tags`
- drafts also carry `target` (linkedin / substack), plus `published` and `url`
  once live
- drafts and ideas also carry `sources`: a list of folder-qualified wikilinks to
  the clips and notes they drew from. `[]` when there are none; never a missing
  key.
- one upward wikilink to the folder's `_index.md`, folder-qualified —
  `[[02-Ideas/_index|↑ Ideas]]`, never a bare `[[_index]]`. Every draft, in
  both subfolders, links to `[[04-Drafts/_index|↑ Drafts]]`: there is no
  `Newsletters/_index.md` and no `LinkedIn-Posts/_index.md`, and there should
  not be. A link to a subfolder index will not resolve.
- **the reciprocal write.** Every clip named in `sources` gets this note appended
  to its own `used-in` list, in the same pass. Append only. Never overwrite the
  list, and never otherwise edit the clip.

**The backlog view.** An idea note written by an inbox sweep also gets a mirrored
row in `notion.content_ideas_db` at Status = Backlog, Platform matching the
qualifier, and the note's path in `Notes`. The file is the source of truth; the
row is the view onto it. An idea note written from a meeting insight does not get
that row — it stands in the folder until synthesis reads it. Clips never get a
row.

**notion.** Destination by kind:

| Kind | Lands in |
|---|---|
| clip | Clips, `content.notion.clips_db` |
| idea | Content Ideas, `notion.content_ideas_db` |
| draft | Content Calendar, `notion.content_db`, at Status = Drafted |

The full text goes in the **page body**; front-matter fields become properties
per the field map below. There is no hub link and no `type` — the database the
row lives in carries both.

The reciprocal write is a relation, and Notion mirrors it: setting Content Ideas
`Source clips` populates Clips `Feeds ideas`, and setting Content Calendar
`Idea source` populates Content Ideas `Calendar post`. **Set one side, then read
the other back to confirm it filled.** Writing both sides by hand is how a
relation ends up doubled.

**There is no mirror on this backend — the row is the note.** An idea written
here lands as one Content Ideas row and nothing else, whether it came from the
inbox sweep or from a meeting insight. This is the one place the two backends
differ in record count rather than in storage, and it is deliberate: the vault's
file-plus-row pair exists because a folder cannot be queried, and a database can.
Do not write a second row to imitate the pair.

**The unrecognized item.** A captured item whose qualifier does not parse goes to
`00-Inbox` on the `vault` backend. There is no inbox on `notion`: it lands as a
Content Ideas row at Status = Backlog, titled as sent, with the parse failure
named in `Notes` so it reads as untriaged rather than as a considered idea. The
convention's rule holds on both — never lose a thought because the user was
typing on a phone.

### 3. Read the metrics log

What published pieces actually did, per piece and per week. Synthesis ranks next
week's angles off this; nothing else in the OS knows what landed.

It also answers **"has this been published already?"** — the rerun check capture
and synthesis both run before proposing an angle. On `vault` that reads
`content.published_log_path` alongside the metrics file; on `notion` it is the
same Posted rows this operation already returns, since the Calendar is the
pipeline and the published log at once. A near-match is flagged in the note or
the proposal, never silently dropped: the user decides whether it is a new angle.

**vault.** `content.metrics_log_path`. Weekly blocks under
`## Week of YYYY-MM-DD`, each holding a table with one row per piece and a bolded
`Read:` takeaway underneath. Read the block for the window asked about, not the
whole file.

Two rules the file's own history earned:

- **`pending` is a value, not a zero.** An owner-only number that has not been
  pulled yet is unknown, and reporting it as zero invents a result.
- **A correction blockquote inside a block outranks the row above it.** Numbers
  here have been restated after a bad pull. Read the correction.

**notion.** Per-piece numbers are properties on the Content Calendar row:
`Impressions`, `Reactions`, `Comments`, `Reposts`, `Profile views`, `Opens`.

```sql
SELECT Title, Platform, "date:Post date:start", Impressions, Reactions,
       Comments, Reposts, "Profile views", Opens
FROM "collection://{notion.content_db}"
WHERE Status = 'Posted' AND "date:Post date:start" >= <window start>
```

An empty analytics property is the `pending` state — the same rule in the shape
the database gives it. The weekly `Read:` narrative has no property: it lives in
that week's wrap page under `notion.week_plans_page`, which is where friday-wrap
already writes it. No separate metrics log survives on this backend.

### 4. Append to a log

Three logs, one operation: the **published log**, the **metrics log**, and the
**backlog**.

**vault.** Append to the named file — `content.published_log_path`,
`content.metrics_log_path`, `content.backlog_path`. Read the last row first and
match the file's existing column order exactly. Keep the front matter and the
trailing sections intact. Idempotency key: the live URL (published log), the
week-of date (metrics), the piece title (backlog). Prior rows are rewritten only
to correct a value the user has supplied, and the report names the before and the
after.

**notion.** There is no append. Each of these logs is a query over rows, so
appending a line is **upserting the row that line described**:

| Log line | Becomes |
|---|---|
| published log row | a Content Calendar row at Status = Posted, with `Live URL`, `Post date`, `Platform`, `Series` |
| metrics row | the analytics properties on that same Calendar row |
| backlog entry | a Content Ideas row at Status = Backlog |

Find the row before creating one — fuzzy-match the title, then any Drafted or
Scheduled row near the date. Patch what you find; create only when nothing
matches. **The connector cannot delete or archive**, so a wrong create is
permanent: propose every batch, keep batches to 10–15 rows so they can be
eyeballed, and give any row written for a test a `[TEST]` title prefix at
creation so it can never be mistaken for content.

### 5. List drafts

Unpublished work in progress, with its target platform, plus the next newsletter
issue number.

**vault.** Files in `content.newsletter_drafts_path` and
`content.linkedin_drafts_path` whose front matter reads `status: drafted`.
Naming: `Issue 0N - <Title>.md`, `Brief - <Title>.md`, `Framework - <Title>.md`,
`Post - <Title>.md`. Target comes from the `target` front-matter key; the folder
is the fallback, never the authority. Next issue number is the highest N in the
newsletter folder plus one — count published and archived issues too, since a
published issue still owns its number.

**notion.** Content Calendar rows at Status = Drafted; `Platform` is the target
and the draft copy is the page body.

```sql
SELECT Title, Platform, Series, "date:Post date:start"
FROM "collection://{notion.content_db}"
WHERE Status = 'Drafted'
```

Next issue number: the highest "Issue #N" among Substack titles **at any
Status**. Filtering to Drafted here reuses a number that is already published.

### 6. Update front matter

Status and provenance on something that already exists. This is the operation
that keeps the trackers from drifting.

**vault.** Edit the YAML block in place and leave the body alone. Add a missing
key rather than rewriting the block. The list-valued provenance keys — `sources`,
`used-in` — are **append-only**: appending is the only edit a skill makes, and
removing a link is the user's decision.

A `status` change also propagates to the item's backlog view, when it has one:
the mirrored Content Ideas row moves to the mapped Status in the same pass, and
is proposed in the same breath as the file edit. A file that says `active` beside
a row that still says `Backlog` is exactly the drift the pair was meant to
prevent. On `notion` there is no second record to propagate to.

**notion.** Set the matching property per the field map. Relations are the
list-valued keys, and **writing a relation replaces it** — read the current
value, then write the union. A relation set from scratch silently drops every
link already on the row.

Status values map like this:

| Vault `status` | Content Ideas | Content Calendar |
|---|---|---|
| `inbox` | Backlog | Idea |
| `active` | Drafting | Idea |
| `drafted` | Drafting | Drafted |
| `published` | Promoted | Posted |
| `archived` | Dropped | Archived |

---

## The field map

| Front matter | Notion home |
|---|---|
| `type` | implied by which database the row lives in |
| title / H1 | `Name` (Clips) · `Idea` (Content Ideas) · `Title` (Content Calendar) |
| `created` | `Captured` (Clips); elsewhere Notion's own created time |
| `clipped` | `Captured` |
| `status` | `Status`, per the mapping above |
| `target` | `Platform` |
| `published` | `Post date` |
| `url` | `Live URL` |
| `source` (clip) | `URL` |
| `Why:` body line | `Why` |
| `tags` | `Theme` (Content Ideas) · `Hashtags` (Content Calendar) |
| `sources` | `Source clips` (Ideas → Clips) · `Idea source` (Calendar → Ideas) |
| `used-in` | `Used in` (Clips → Calendar) · `Feeds ideas` (Clips → Ideas) |
| hub wikilink | no equivalent; the parent page and the relations carry navigation |
| note body | the page body |

`sources` and `used-in` are the same fact written from both ends. Either one
alone decays: a draft that names its clips in prose loses them when the session
closes, and a clip with no return link is indistinguishable from dead weight at
prune time. The pair is what makes "did this clip ever earn its keep" a query
instead of a guess. On the `notion` backend the pair is a relation and its
mirror — the same discipline with the bookkeeping handled.

## The Clips database

Config key `content.notion.clips_db`. Title property: **Name**. Full clip text
and any pull-quotes live in the page body.

| Property | Type | Options / notes |
|---|---|---|
| Name | title | — |
| URL | url | the dedupe key: one URL is one clip |
| Source | select | Substack (orange), LinkedIn (blue), Web (gray) |
| Why | text | the one-line reason it was kept |
| Captured | date | — |
| Used in | relation → Content Calendar | the `used-in` half of provenance |
| Feeds ideas | relation → Content Ideas | the `sources` half, from the clip side |

`Source` preserves the only information the vault's subfolders carried.
`Feeds ideas` is what keeps provenance bidirectional — without it a clip can say
what it became but not what idea it fed. Content Ideas gains one matching
relation, `Source clips` → Clips, which auto-creates `Feeds ideas` on this side.

## Beyond the six

Two things that are not storage operations but still differ by backend. Skills
name them the same way they name the six; the branch lives here.

### The theme map

One named document rather than a store, read and rewritten whole: the file at
`content.theme_map_path` on the `vault` backend, the page at
`content.notion.theme_map_page` on `notion`. Same content, same discipline —
`wednesday-synthesis` reads it for the forward roadmap and updates it when a
theme is picked, and `monday-planner` reads it for the week's roadmap slot.

### The content hygiene check

The weekly integrity pass, run by `friday-wrap`. Report each result as a
one-liner — what passed, what needs a fix. Propose fixes; apply only on
confirmation.

**vault.** Run the checklist at `content.vault_health_checklist` and report it
item by item.

**notion.** The checklist's items are folder hygiene for a store that no longer
has folders. Three queries replace it, and each should return zero rows.

1. Posted with no live link — a publish that never got marked, or a mark that
   half-ran:

```sql
SELECT Title, "date:Post date:start" FROM "collection://{notion.content_db}"
WHERE Status = 'Posted' AND ("Live URL" IS NULL OR "Live URL" = '')
```

2. A graduation that lost its link:

```sql
SELECT Idea FROM "collection://{notion.content_ideas_db}"
WHERE Status = 'Promoted'
  AND ("Calendar post" IS NULL OR "Calendar post" = '' OR "Calendar post" = '[]')
```

3. Prune candidates — clips past the lookback window that never fed anything:

```sql
SELECT Name, Captured FROM "collection://{content.notion.clips_db}"
WHERE Captured < <lookback start>
  AND ("Used in" IS NULL OR "Used in" = '' OR "Used in" = '[]')
  AND ("Feeds ideas" IS NULL OR "Feeds ideas" = '' OR "Feeds ideas" = '[]')
```

**Query 3 is not a failure.** It is a list to review, capped at ten rows in the
wrap output. Rows above that cap are counted, not listed.

**Clips are the one item to be careful with, on either backend.** The keep test
is provenance, not age: a clip that fed a draft stays however old it is. On
`vault`, stale unused clips are **staged** into `_to_delete/clips-YYYY-MM-DD/`,
never deleted. On `notion` nothing can be deleted at all (the connector has no
delete or archive), so a retired clip is tombstoned with a `[DROPPED]` title
prefix and cleared by hand later. Never prune in bulk without naming every item
first, and if a prune would touch more than a quarter of the clip library in one
pass, stop and hand the decision back rather than proposing it as routine
maintenance.

## Config keys

```
content.backend                  "vault" | "notion"
content.notion.clips_db          <clips data source id>      notion backend only
content.notion.theme_map_page    <theme map page id>         notion backend only
content.vault_root + the nine other path keys                vault backend only
content.default_post_count       backend-neutral
content.synthesis_lookback_days  backend-neutral
content.polish.*                 backend-neutral
content.metrics.*                backend-neutral
capture.idea_capture.*           backend-neutral in shape; the notes it produces
                                 route through "write a note"
firewall.walled_engagements      enforced by every operation, on both backends
```

## Config missing

- **No `content` block at all.** The OS runs without a content pipeline. Skip
  every content step silently — no error, no mention. This is existing behavior
  and it does not change.
- **A `content` block with no `content.backend`.** Stop and say so rather than
  assuming one:

  "content.backend is not set in solo-os-config.json. Set it to "vault" or
  "notion" before running a content skill."

  Guessing is the half-branch this contract exists to prevent, and a wrong guess
  writes into a store the user is not looking at.
- **`content.backend` set to `"notion"` with `content.notion.clips_db` empty.**
  Stop and say so. Run onboarding to provision the Clips database.
