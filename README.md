# brooksby-ops

The Solopreneur OS, packaged as a Cowork plugin. Config-driven agents for a solo consultant who runs the business out of Notion, Gmail, Google Calendar, and Granola, with an optional local markdown vault behind the content engine.

The skills carry the logic. Your data lives in one config file. To run this for a different person, you swap the config, not the skills.

**Version 0.11.6** — thirteen skills. See `## Status` for what changed.

## What's in here

### Business operations

| Skill | What it does |
|---|---|
| `onboarding` | First-time setup. Provisions the seven Notion databases, captures their IDs into config, connects the tools, and verifies with a check-in. Say "set up the OS". |
| `daily-checkin` | The whole morning in one command, three phases in order: runs `capture` to pull in new meetings and emailed ideas, shows the today view (meetings with prep status, due and overdue work, email triaged into proposed tasks, deliverable check-off), then refreshes the Daybook last so it reflects the writes. `checkin.run_capture` and `checkin.render_daybook` turn the outer phases off. Say "check in". |
| `monday-planner` | Builds the week plan page in Notion, creates prep tasks, captures last week's meetings, drafts the Monday weekly email. With a content vault configured, the plan carries the standing content deliverables with their draft paths. Say "plan my week". |
| `capture` | Two independent sweeps. Ideas: self-addressed clip and idea emails swept from Gmail into the content vault, Notion Content Ideas, and Agent Ideas, deduped by a processed label. Meetings: recent Granola meetings into Notion meeting pages with minutes, action items, and status, plus a Word copy to the engagement folder for client-facing meetings and a sanitized idea note when a meeting surfaces a teachable insight. Runs standalone and as a step inside the check-in and planner. Gmail drives the idea sweep, Granola the meeting sweep; either runs without the other. |
| `friday-wrap` | Closes out the week: confirms what got done, rolls overdue open work forward as carryover, writes a week-wrap page. With a content vault configured, unpublished drafts roll as carryover and the vault-health checklist runs with the wrap. Say "weekly wrap". |
| `build-prep` | On-demand prep brief for a named or next meeting: last-meeting status, open items owed, recent email, and agenda in one place. Say "build the prep". |
| `render-daybook` | One-page snapshot of what is true right now: today's schedule, due and waiting work, work by engagement, the deliverable radar, and optional revenue. Reads only; publishes a Cowork artifact. Runs standalone and as the last phase of the check-in. Say "refresh my daybook". |

### Content engine

| Skill | What it does |
|---|---|
| `wednesday-synthesis` | Reads recent clips, ideas, and metrics and returns three ranked newsletter angles with the LinkedIn posts each would derive. Brian picks; the skill never chooses. Updates the forward theme map. Say "run my synthesis". |
| `draft-content` | Drafts the newsletter issue, then cuts LinkedIn posts from it. Hard gate between the two phases: the issue is finalized before any post exists. Side-by-side variants, losers archived not deleted. Say "draft issue #N on [angle]". |
| `polish-and-score` | The quality gate. Two chains by asset type: Substack gets voice, word humanizer, newsletter-voice, and an anti-slop audit; LinkedIn gets voice, structure humanizer, word humanizer, audit, and a score against real performance data. Say "polish and score". |
| `surface-humanizer` | The word-level pass, run as a repeatable seven-stage checklist against `anti-ai-writing.md`: banned words, banned phrases, banned structures, punctuation, sentence starters, rhythm, and the four-question filter. Pairs with the account-level `stay-human`, which fixes structure. |
| `mark-published` | One command syncs all four trackers the moment a piece goes live: Notion Content Calendar, Content Ideas, vault `published-posts-log.md`, and vault `content-backlog.md`. Idempotent. Say "mark issue #N published". |
| `collect-metrics` | Pulls the public LinkedIn counts, prompts for the owner-only numbers and the Substack figures, and writes both trackers together. Matches posts on the numeric URN, not the URL string. Say "collect my metrics". |

## Layout

```
brooksby-ops/
  .claude-plugin/
    plugin.json                              # plugin manifest
  skills/
    onboarding/
      SKILL.md                               # first-time setup
      references/notion-schema.md            # the seven-database schema spec
    daily-checkin/SKILL.md
    capture/SKILL.md
    friday-wrap/SKILL.md
    build-prep/SKILL.md
    render-daybook/
      SKILL.md
      references/connector-health.md         # connector probes and failure wording
    monday-planner/
      SKILL.md
      references/weekly-email.md             # generic vs per-engagement email setup
    wednesday-synthesis/SKILL.md
    draft-content/SKILL.md
    polish-and-score/SKILL.md
    surface-humanizer/SKILL.md
    mark-published/SKILL.md
    collect-metrics/SKILL.md
  references/
    engagement-routing.md                    # every task gets an engagement; the four internal buckets
  templates/
    weekly_email_TEMPLATE.html               # generic weekly email, copy per engagement
  config.example.json                        # sanitized template (onboarding writes the real one)
  .gitignore                                 # keeps the real config and About Me/ out of the repo
  README.md
```

## Install

Onboarding does the setup for you. You do not hand-fill database IDs.

1. Add this plugin to Cowork (Customize > Plugins).
2. Connect Notion, Gmail, and Google Calendar in Cowork. Granola is optional but required for `capture`.
3. In Notion, share the page you'll use as the OS home with the Notion integration, so the plugin can create databases under it.
4. Pick the Cowork project folder where the OS will live. This is where your `solo-os-config.json` gets written.
5. Say **"set up the OS"** to run onboarding. It creates the seven Notion databases, builds your home dashboard, writes the config, and verifies the install with a check-in.

After that, say "check in" for the daily view or "plan my week" for the Monday planner.

When upgrading, delete any stale standalone copies of the content skills that lack the `brooksby-ops:` prefix. An old copy will otherwise trigger alongside the new one.

## How config works

Every skill finds `solo-os-config.json` by searching your connected Cowork project folder(s) by filename before it runs. No fixed path is assumed, so it works whatever you named the folder. If the file is missing, or a required key is absent, the skill stops and points you to onboarding. No instance data — IDs, emails, paths, customer names, rates — is ever hardcoded in a skill.

The real `solo-os-config.json` is git-ignored and never ships. Only the sanitized `config.example.json` is committed.

One optional block worth knowing about: `content` (shape in `config.example.json` under `_content_example`). Add it to hook the OS into a local content vault. The planner lists the standing content deliverables, capture writes sanitized idea notes under the vault's output contract, the wrap rolls unpublished drafts and runs the vault-health checklist, and the six content-engine skills come alive. Leave it out and the OS runs as a business-operations stack only. Two rules always hold: idea notes are sanitized (no client names, figures, or confidential detail), and engagements in `firewall.no_connector_accounts` never feed content.

## Two things that live outside this plugin

Both are account-level skills the content chain calls, and neither can be patched by repackaging this plugin.

- **`stay-human`** is the structure humanizer. Its own rules specify British English. Brian writes American English, so `polish-and-score` carries an explicit override and a spelling sweep after every structure pass.
- **`post-scorer`** is the LinkedIn scorer. It hardcodes its Apify actor at step 2 and does not read `content.metrics.apify_linkedin_actor`. Change that config key and the scorer keeps using its literal until someone edits the skill. The config file carries a note on the key saying so.

## Status

**v0.11.6 (2026-08-15)** — the check-in becomes the whole morning.

- `daily-checkin` now runs in three explicit phases: **capture**, then the **today view**, then **render-daybook**. One command instead of three.
- The order is load-bearing. Capture writes, the today view reads and writes, the Daybook renders last as a read layer over both. Rendering the Daybook first produced a snapshot that was stale before the user saw it.
- Phases 1 and 2 share **one confirmation gate**. Approving a morning should not take three rounds.
- New config block `checkin` — `run_capture` and `render_daybook`, both defaulting to true when absent. Set either false for the bare today view.
- **The email split is now explicit.** Both phases read Gmail, so the skill states which one owns what: mail matching the idea-capture convention becomes a note via capture step 0, everything else becomes a task via the today view. The today view excludes the capture processed label, so an emailed idea can no longer become both.
- The old "Meetings to capture" output block is gone — phase 1 does that work now. It is replaced by a one-line **Captured** summary.
- `capture` and `render-daybook` remain standalone commands. This release adds a caller, it does not remove an entry point.

**v0.11.5 (2026-08-15)** — capture gains an email idea sweep.

- `capture` is now two independent sweeps. New **step 0** queries Gmail for self-addressed clip and idea mail, parses the subject grammar, and routes each item to the content vault, Notion Content Ideas, or the Agent Ideas DB. The meeting sweep is unchanged and unaffected; either runs without the other.
- New shared reference `references/idea-capture-convention.md` holds the subject grammar (`<Verb> <qualifier>: <title>`), the routing table, the `Why:` body rule, legacy subject forms, and the dedupe rule. The skill points at it rather than restating it.
- New config block `capture.idea_capture` — `processed_label`, `lookback_days`, `personal_ref_path`. The processed label is the dedupe key and is applied only after a note is written, so a failed run never loses a thought.
- `personal_ref_path` is deliberately outside the vault output contract. It is a personal reference shelf, not a content source.
- The gap this closes: `wednesday-synthesis` consumed `01-Clips` and `02-Ideas`, but the only producers were the web clipper and hand-typed notes. Mail sent from a phone had no path into the system and silently piled up in the inbox.

**v0.11.4 (2026-08-15)** — plans and wraps get their own parent page, and content provenance runs both directions.

Two unrelated leaks, fixed together. The week plan and the week wrap were each written as sub-pages of the home page, so they never sat next to each other and the home page collected loose week pages. Separately, a clip that fed a draft was named in chat and nowhere else — the vault held 43 clips with zero backlinks, because nothing ever wrote the return link.

- New config key **`notion.week_plans_page`**: the shared parent for both. `monday-planner` and `friday-wrap` now require it and write their pages under it, in date order. If the key is absent both fall back to `notion.home_page` and say so in the output, so an older config degrades instead of failing.
- `onboarding` gains **step 2.6**: it creates the "Weekly Plans and wraps" page on a fresh install and writes its ID to config. Verification step 3 confirms the key resolves.
- `wednesday-synthesis` names supporting material as folder-qualified wikilinks, not prose, and on Brian's pick writes that cluster into the idea note's frontmatter as a **`sources:`** list. That list is the handoff. An angle with no feeding clip carries `sources: []`, never a missing key, so "no source" stays distinguishable from "never recorded."
- `draft-content` inherits `sources:` onto every asset it creates, then stamps the return link — **`used-in:`** — onto each source clip. Append only: this is the one circumstance in which the skill writes into `01-Clips`, and it touches nothing but that key. Runs at the end of Phase 1 for the finalized issue and again in Phase 2 for each derived post. Losing variants stamp nothing.
- That closes a loop `friday-wrap` was already half of: its vault-health prune uses `used-in:` as the clip keep test, so until something wrote the key, every clip looked unused and age was the only signal available. Synthesis writes `sources:`, draft stamps `used-in:`, the wrap reads it.
- The engagement audit grew from one query to three, in `references/engagement-routing.md`: open Tasks with a null Engagement, **Meeting pages** with a null Engagement, and **duplicate meeting pages grouped on `Granola link`**. Each passes while the others fail. Capture writes an engagement to three record types, so checking Tasks alone leaves meeting pages orphaned; and a failed dedupe re-summarizes the call from scratch, so the copies carry different titles and no title- or date-based check will ever find them. Two cautions came out of the 2026-08-11 incident: scan the whole table rather than a date window, and never bulk-delete on the "empty copy is the duplicate" heuristic alone.
- `onboarding` step 5, `onboarding/references/notion-schema.md`, and `capture` step 2 all point at the three-query set.

No schema version change — config stays **1.3**, with one added key.

**v0.11.3 (2026-08-11)** — every task belongs to an engagement.

Nothing stopped a skill writing a task with a blank Engagement, so non-client work — content, admin, prospecting, coffee — piled up untagged. The work-by-engagement tile filled with an Unassigned group and the client rollups were incomplete.

Engagements now holds four internal buckets alongside the client rows: **Marketing Content**, **Business Admin**, **Business Development**, and **Networking**. They carry a title and an icon and nothing else — no Status, Rate, Billing model, or Start date — which is exactly what keeps them out of the `Status = Active` views and revenue reporting.

- New shared reference `references/engagement-routing.md` holds the convention, the bucket boundaries, and the ask-don't-guess rule. `daily-checkin`, `monday-planner`, `build-prep`, and `capture` all point at it rather than restating it.
- `build-prep` and `capture` drop the "or none" escape when asking which engagement, and offer the four buckets alongside the client list. In capture a null propagated into three records, so this was the expensive one.
- `monday-planner` no longer leaves prep-task engagements blank when a client is not identifiable, and the standing content deliverables now land on Marketing Content.
- `render-daybook` orders client groups above internal buckets so a content group cannot outrank a client due the same day. The Unassigned group now reads as a defect, not a category.
- `onboarding` gains step 2.4: it seeds the four buckets on a fresh install and writes their page IDs to config, so a new user cannot create orphans on day one. Verification adds a null-Engagement query that must return zero.
- Config schema **1.3**: new `notion.internal_engagements` block. Required key for the four task-creating skills; optional for the Daybook, which degrades to due-date-only ordering without it.

Routing rule worth knowing: **client invoicing follow-through links to the client engagement, not Business Admin.** Chasing an invoice is client work. Business Admin holds money work with no client behind it — taxes, subscriptions, the accounting tool.

Also in this release: `render-daybook` finally appears in the skill table and layout above. It shipped earlier and was never documented here.

Not in this release: the recurring `Content:` cadence tasks are still seeded by hand. Folding that into `monday-planner` is the stated intent and is its own change.

**v0.11.2 (2026-08-05)** — prep briefs move into the prep task, and capture links them back.

The prep brief used to land as a standalone Notion page, separate from the Prep task that tracked it, and nothing ever connected either one to the meeting page capture wrote afterward. Three artifacts per meeting, none of them linked. Rescheduling made it worse: a rebuilt brief left the old page sitting there, so two prep documents claimed the same meeting.

- `build-prep` step 5 rewritten. The prep task **is** the brief: the full text goes in the task's page body, replacing rather than appending so a rebuild never stacks on a stale draft. No standalone prep page, ever. No Word copy unless Brian asks for one to send to someone.
- `capture` gains **step 5d**. Before proposing a meeting page it looks for a matching Prep task, links it via the `Meeting` relation, sets the engagement, marks it Done, and prepends a `**Prep for this meeting:**` line above the Summary. The relation is bidirectional, so the plan and the record each point at the other.
- Both skills now name stale duplicate prep artifacts in the disposition instead of leaving them to rot.
- New hard rule in capture: never leave a prep artifact unlinked when its meeting page exists.

No schema change. `Tasks.Type = Prep` and `Tasks.Meeting` already existed; they were just never wired together.

**v0.11.1 (2026-07-28)** — fix release off the nine-point end-to-end test. Seven defects closed:

1. `collect-metrics` matched LinkedIn posts by URL string. LinkedIn issues each post both a `share_urn` and an `activity_urn`, and the Calendar stores whichever form was captured at publish time, so string matching found one of three posts and reported success. Now matches on the trailing numeric URN against both forms, and reports "matched N of M" every run. Same step gained the warning against passing a `fields` projection to Apify `get-dataset-items`, which silently strips the nested `stats` object and returns all zeros.
2. `voice.newsletter_voice_path` pointed at a file that did not exist, and `polish-and-score` was instructed to regenerate it. That would have quietly discarded a profile tuned across two published issues. The file now sits in `About Me/` with the other voice files and the skill is forbidden from regenerating it. **It is deliberately NOT in `voice.sources`** — it is a Substack-only register and would bleed into LinkedIn drafting if treated as a general source; it is reached only through `newsletter_voice_path`. *(This line claimed the opposite until 2026-08-28. Both configs have always said the correct thing; the README was the copy that drifted.)*
3. `surface-humanizer` was named in config but had never been built. Now shipped as a skill.
4. `draft-content` specified file names and hub links that did not match the vault. It produced `issue-N_slug.md` against an on-disk convention of `Issue 0N - Title.md`, and linked to two `_index.md` files that do not exist. Now matches the vault, and every derived post links back to its source issue.
5. `stay-human` drifts drafts into British spellings against an American-English profile. `polish-and-score` now carries the override as a hard rule plus an explicit spelling sweep.
6. `post-scorer` hardcodes its Apify actor while `collect-metrics` reads config. Documented on both sides.
7. `mark-published` could not correct a Hook that went stale when Brian edited copy at publish time. The rule is narrowed: never overwrite with an inferred line, but do overwrite when the actual published copy is supplied, and report the change.

Also in this release: the Content Calendar gains an **Archived** status, for a piece killed rather than deferred. Schema reference updated.

**v0.11.0 (2026-07-23)** — the content engine ships. Five skills (`wednesday-synthesis`, `draft-content`, `polish-and-score`, `mark-published`, `collect-metrics`) turn the vault and the Notion content pair into a weekly loop: synthesize Wednesday, draft and cut, polish and score, publish and sync four trackers, collect metrics Friday, and feed next Wednesday's synthesis.

**v0.10.0** — optional content vault hooks. A `content` config block wires the planner, capture, and the Friday wrap into a local markdown vault. Notion owns tasks and calendar; the vault owns drafts and research.

**v0.9.0** — the content pair becomes a full LinkedIn/Substack workflow. Content Ideas gains Hook and Theme; Content Calendar becomes the post pipeline and the published log.

**v0.8.0** — seventh database, Content Ideas.

Verified: onboarding dry-run against the live Notion connector (databases and two-way relations build cleanly in a fresh workspace, check-in reads end to end, home dashboard builds programmatically). Nine-point end-to-end content test passed 2026-07-28.

Remaining before beta: one unattended full-flow test with a real tester, and re-verification of the seven fixes above after reinstall.
