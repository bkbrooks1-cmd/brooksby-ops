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
Used if present: `content.metrics.apify_linkedin_actor`, `content.metrics.linkedin_profile_url` (the actor's input; if it is absent, say so and ask once rather than guessing a handle), `content.metrics.linkedin_auto_fields`, `content.metrics.linkedin_manual_fields`, `content.metrics.substack_manual_fields`.

Note: `content.metrics.apify_linkedin_actor` is read by this skill only. The account-level `post-scorer` skill hardcodes the same actor at its step 2 and does not read config. Change the config key and post-scorer keeps using its literal. Keep the two in sync by hand until post-scorer is brought into the plugin.

## What can and cannot be automated (be honest about this)

- **LinkedIn — public, automatable:** reactions, comments, reposts. These show on the public post and can be pulled with an Apify actor.
- **LinkedIn — private, manual:** **impressions and profile views are owner-only** — LinkedIn shows them only to Brian under "View analytics" on each post. No compliant scraper reads them. Prompt Brian.
- **Substack — manual:** Substack has no official public API. Opens, open rate, and clicks live in the dashboard; subscriber totals are exportable as CSV. Prompt Brian, or read a CSV he drops in.

Never present a scraped or guessed number as if it were the private analytics. If Apify isn't configured or fails, collect everything by prompt instead and say so.

## The routine

1. **Find this week's posted pieces.** Query the Content Calendar (`collection://{notion.content_db}`) for rows with `Status = Posted` and a `Post date` in the last 7 days (or the window Brian names). Collect their `Live URL`s.

2. **Auto-pull the public LinkedIn counts.** If `content.metrics.apify_linkedin_actor` is set, run it against `content.metrics.linkedin_profile_url` and pull the full result set. If that key is missing, ask for the profile URL once and say it belongs in config. Then match actor results to Calendar rows **by numeric URN, never by URL string** — see below. If the actor is not set or the run fails, note it and fall back to prompting.

   **The matching rule (this is where it breaks silently).** LinkedIn issues every post two different numeric IDs: a `share_urn` and an `activity_urn`. They are not the same number. The Content Calendar stores whichever form was captured at publish time and both forms are present in the data today. The actor returns `activity_urn` URLs. Comparing URL strings therefore finds only the rows that happen to hold the activity form, and the run still reports success.

   Do this instead:
   - Extract the numeric ID from each Calendar `Live URL`. Three forms are in the data and all three must parse:
     - `.../posts/<vanity>_<slug>-activity-7486101337173057536-dbl8/` — digits after `activity-`
     - `.../posts/<vanity>_<slug>-share-7486101334044225537-dbl8/` — digits after `share-`
     - `.../feed/update/urn:li:activity:7486101337173057536/` — digits after the last colon
     In the first two forms a short alphanumeric token follows the ID (`-dbl8`, `-XiZt`, `-st5T`, `-n_bz`). It is not part of the ID. The safe rule: take the **longest run of consecutive digits** in the URL, which is the 19-digit URN in every form seen so far.
   - For each actor result, read **both** `urn.activity_urn` and `urn.share_urn`.
   - A Calendar row matches an actor result when its numeric ID equals **either** value.

   Same post under both forms:
   ```
   share_urn     7486101334044225537   <- what the Calendar row holds
   activity_urn  7486101337173057536   <- what the actor returns
   ```

   Measured 2026-07-28: three LinkedIn posts published that week, two stored as `share-` and one as `activity-`. URL-string matching found one of three and reported success on a third of the data.

   **Never pass a `fields` projection to Apify `get-dataset-items`.** It silently strips the nested `stats` object and every count returns 0. The run looks clean and the numbers are all zeros. Request full items and pick the fields out yourself. (The account-level `post-scorer` skill carries the same warning at its step 2.)

3. **Report the match count before writing.** State "matched N of M posted rows" and name every row that did not match, with its stored URL. A partial match is a result to be reported, never a result to be rounded up.

4. **Prompt for the private numbers**, batched in one clean list so Brian fills them fast:
   - Per LinkedIn post: impressions, profile views (from "View analytics").
   - Per Substack issue: opens, open rate, new subscribers (from the Stats page) — or accept a subscriber CSV.

5. **Write to both trackers, together:**
   - **Content Calendar row** (per piece): set `Impressions`, `Reactions`, `Comments`, `Reposts`, `Profile views` (LinkedIn) and `Opens` (Substack) from what was gathered; add a one-line `Performance notes` read.
   - **`metrics-log.md`** (`content.metrics_log_path`): append this week's block in the file's existing format — one row per piece with the same numbers — under a `## Week of YYYY-MM-DD` heading, plus a one-line "Read:" takeaway (what beat what, and why).

6. **Report** every write with links, and name anything still missing (e.g. a post whose 7-day window hasn't closed) so it gets picked up next week.

## Hard rules

- **Never match LinkedIn posts by URL string.** Always match on the trailing numeric URN, checked against both `urn.activity_urn` and `urn.share_urn`. Two different numbers, one post.
- **Never pass `fields` to Apify `get-dataset-items`.** It strips nested `stats` and returns zeros without raising an error.
- **Report the match count.** "Matched N of M" every run. If N is less than M, name the misses.
- **Never fabricate a metric.** Public counts come from the actor; private numbers come from Brian. If a source is unavailable, say so and leave the field blank.
- **A zero is suspect.** If every count in a run returns 0, treat it as a failed pull, not a real result. Check the `fields` projection first.
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
