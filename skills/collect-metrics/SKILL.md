---
name: collect-metrics
description: Gather this week's content metrics and write them to every tracker at once. Use when Brian says "collect my metrics", "collect this week's metrics", "log the numbers", or as the metrics step inside the Friday wrap. Auto-pulls the LinkedIn counts that are public, prompts Brian for the private ones and the Substack numbers, then writes them to the Notion Content Calendar rows and the vault metrics-log together.
---

# Collect metrics

Turn the weekly metrics chore into one pass. The honest constraint: some numbers are public (a scraper can read them) and some are private to Brian (visible only when he's logged in). This skill automates the public half, prompts for the private half, and writes everything to both trackers so the numbers never live in only one place. Next Wednesday's synthesis reads `metrics-log.md` — that's the loop closing.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before collecting metrics."

Required keys: `notion.content_db`, `content.metrics_log_path`.
Used if present: `content.metrics.apify_linkedin_actor`, `content.metrics.linkedin_auto_fields`, `content.metrics.linkedin_manual_fields`, `content.metrics.substack_manual_fields`.

## What can and cannot be automated (be honest about this)

- **LinkedIn — public, automatable:** reactions, comments, reposts. These show on the public post and can be pulled with an Apify actor.
- **LinkedIn — private, manual:** **impressions and profile views are owner-only** — LinkedIn shows them only to Brian under "View analytics" on each post. No compliant scraper reads them. Prompt Brian.
- **Substack — manual:** Substack has no official public API. Opens, open rate, and clicks live in the dashboard; subscriber totals are exportable as CSV. Prompt Brian, or read a CSV he drops in.

Never present a scraped or guessed number as if it were the private analytics. If Apify isn't configured or fails, collect everything by prompt instead and say so.

## The routine

1. **Find this week's posted pieces.** Query the Content Calendar (`collection://{notion.content_db}`) for rows with `Status = Posted` and a `Post date` in the last 7 days (or the window Brian names). Collect their `Live URL`s.
2. **Auto-pull the public LinkedIn counts.** For each LinkedIn URL, if `content.metrics.apify_linkedin_actor` is set, run it to get reactions / comments / reposts. If it's not set or the run fails, note that and fall back to prompting.
3. **Prompt for the private numbers**, batched in one clean list so Brian fills them fast:
   - Per LinkedIn post: impressions, profile views (from "View analytics").
   - Per Substack issue: opens, open rate, new subscribers (from the Stats page) — or accept a subscriber CSV.
4. **Write to both trackers, together:**
   - **Content Calendar row** (per piece): set `Impressions`, `Reactions`, `Comments`, `Reposts`, `Profile views` (LinkedIn) and `Opens` (Substack) from what was gathered; add a one-line `Performance notes` read.
   - **`metrics-log.md`** (`content.metrics_log_path`): append this week's block in the file's existing format — one row per piece with the same numbers — under a `## Week of YYYY-MM-DD` heading, plus a one-line "Read:" takeaway (what beat what, and why).
5. **Report** every write with links, and name anything still missing (e.g. a post whose 7-day window hasn't closed) so it gets picked up next week.

## Hard rules

- **Never fabricate a metric.** Public counts come from the actor; private numbers come from Brian. If a source is unavailable, say so and leave the field blank.
- **Same-sync principle as mark-published:** Calendar and metrics-log update together, never one without the other.
- **No client names in the vault.** OFP never enters `D:\Brain`.
- Confirm before writing Notion; write the vault log under the output contract.

Output contract (D:\Brain). Every note you create or edit in the capture
folders (00-Inbox, 01-Clips, 02-Ideas, 03-Claude-Sessions, 04-Drafts):
- frontmatter: type (clip|idea|draft|session|reference|hub),
  created (YYYY-MM-DD), status (inbox|active|drafted|published|archived), tags
- one upward wikilink to the folder's _index.md, folder-qualified:
  [[02-Ideas/_index|↑ Ideas]] — never a bare [[_index]]
Never write into _ref-* folders or project junctions. Never touch OFP.
Never leave a note without frontmatter or a hub link.
