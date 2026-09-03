---
name: collect-metrics
description: Gather this week's content metrics and write them to every tracker at once. Use when the user says "collect my metrics", "collect this week's metrics", "log the numbers", or as the metrics step inside the Friday wrap. Auto-pulls the LinkedIn counts that are public, prompts the user for the private ones and the Substack numbers, then writes them to the Notion Content Calendar rows and the metrics log together.
---

# Collect metrics

Turn the weekly metrics chore into one pass. The honest constraint: some numbers are public (a scraper can read them) and some are owner-only (visible just to the account holder, logged in). This skill automates the public half, prompts for the private half, and writes everything to every tracker so the numbers never live in only one place. Next Wednesday's synthesis reads the metrics log — that's the loop closing.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before collecting metrics."

Required keys: `notion.content_db`, `content.backend`.
Used if present: `content.metrics.apify_linkedin_actor`, `content.metrics.linkedin_profile_url` (the actor's input; if it is absent, say so and ask once rather than guessing a handle), `content.metrics.linkedin_auto_fields`, `content.metrics.linkedin_manual_fields`, `content.metrics.substack_manual_fields`.

**Storage.** The metrics log is reached through named operations in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`. The Content Calendar is Notion on both backends — that is the pipeline of record either way, so `notion.content_db` is read directly, not through the contract.

Note: `content.metrics.apify_linkedin_actor` is read by this skill only. The account-level `post-scorer` skill hardcodes the same actor at its step 2 and does not read config. Change the config key and post-scorer keeps using its literal. Keep the two in sync by hand until post-scorer is brought into the plugin.

## What can and cannot be automated (be honest about this)

- **LinkedIn — public, automatable:** reactions, comments, reposts. These show on the public post and can be pulled with an Apify actor.
- **LinkedIn — private, manual:** **impressions and profile views are owner-only** — LinkedIn shows them only to the account holder under "View analytics" on each post. No compliant scraper reads them. Prompt the user.
- **Substack — manual:** Substack has no official public API. Opens, open rate, and clicks live in the dashboard; subscriber totals are exportable as CSV. Prompt the user, or read a CSV they drop in.

Never present a scraped or guessed number as if it were the private analytics. If Apify isn't configured or fails, collect everything by prompt instead and say so.

## The routine

1. **Find this week's posted pieces.** Query the Content Calendar (`collection://{notion.content_db}`) for rows with `Status = Posted` and a `Post date` in the last 7 days (or the window the user names). Collect their `Live URL`s.

2. **Auto-pull the public LinkedIn counts.** If `content.metrics.apify_linkedin_actor` is set, run it against `content.metrics.linkedin_profile_url` and pull the full result set. If that key is missing, ask for the profile URL once and say it belongs in config. Then match actor results to Calendar rows **by numeric URN, never by URL string** — see below. If the actor is not set or the run fails, note it and fall back to prompting.

   **The matching rule (this is where it breaks silently).** LinkedIn issues every post two different numeric IDs: a `share_urn` and an `activity_urn`. They are not the same number. The Content Calendar stores whichever form was captured at publish time and both forms are present in the data today. The actor returns `activity_urn` URLs. Comparing URL strings therefore finds only the rows that happen to hold the activity form, and the run still reports success.

   Do this instead:
   - Extract the numeric ID from each Calendar `Live URL`. Three forms are in the data and all three must parse:
     - `.../posts/<vanity>_<slug>-activity-7486101337173057536-dbl8/` — digits after `activity-`
     - `.../posts/<vanity>_<slug>-share-7486101334044225537-dbl8/` — digits after `share-`
     - `.../feed/update/urn:li:activity:7486101337173057536/` — digits after the last colon
     In the first two forms a short alphanumeric token follows the ID (`-dbl8`, `-XiZt`, `-st5T`, `-n_bz`). It is not part of the ID. The safe rule: take the **longest run of consecutive digits** in the URL, which is the 19-digit URN in every form seen so far.
   - For each actor result, read **all three** of `urn.activity_urn`, `urn.share_urn`, and `urn.ugcPost_urn`. The third is null on most posts but is populated instead of `share_urn` on some — verified live 2026-07-28 on the Fable 5 post, whose `full_urn` is `urn:li:ugcPost:...` and whose `share_urn` is null.
   - A Calendar row matches an actor result when its numeric ID equals **any** of the three.
   - **If more than one actor result matches, disambiguate.** A repost and its original carry the same `share_urn` while having different `activity_urn`s, so a share-form match can return two results (verified live: `share_urn` 7457456510340665344 appears on both a `post_type: regular` item and a `post_type: repost` item). Prefer the result whose `activity_urn` matches exactly; if neither does, prefer `post_type: regular` over `repost`. Never sum the two — that double-counts.

   Same post under both forms:
   ```
   share_urn     7486101334044225537   <- what the Calendar row holds
   activity_urn  7486101337173057536   <- what the actor returns
   ```

   Measured against live actor output 2026-07-28: three LinkedIn posts published that week, two stored as `share-` and one as `activity-`. URN matching found **three of three**. URL-string matching found **zero of three** — worse than it looks on paper, because the actor's URLs carry `?utm_source=...` query strings and a different trailing token than the Calendar's stored copy, so even the row stored in activity form fails a string compare.

   **Never pass a `fields` projection to Apify `get-dataset-items`.** It silently strips the nested `stats` object and every count returns 0. The run looks clean and the numbers are all zeros. Request full items and pick the fields out yourself. (The account-level `post-scorer` skill carries the same warning at its step 2.)

3. **Report the match count before writing.** State "matched N of M posted rows" and name every row that did not match, with its stored URL. A partial match is a result to be reported, never a result to be rounded up.

4. **Prompt for the private numbers**, batched in one clean list so the user fills them fast:
   - Per LinkedIn post: impressions, profile views (from "View analytics").
   - Per Substack issue: opens, open rate, new subscribers (from the Stats page) — or accept a subscriber CSV.

5. **Write to every tracker, together:**
   - **Content Calendar row** (per piece): set `Impressions`, `Reactions`, `Comments`, `Reposts`, `Profile views` (LinkedIn) and `Opens` (Substack) from what was gathered; add a one-line `Performance notes` read.
   - **The metrics log** — **append to a log**: this week's entry, one row per piece with the same numbers, plus a one-line "Read:" takeaway (what beat what, and why). The operation owns the format and says where the entry and the takeaway land on this backend; on `notion` they are the row you just wrote and the week's wrap page, so this is not a second copy of the numbers.

6. **Report** every write with links, and name anything still missing (e.g. a post whose 7-day window hasn't closed) so it gets picked up next week.

## Hard rules

- **Never match LinkedIn posts by URL string.** Always match on the numeric URN, checked against `urn.activity_urn`, `urn.share_urn`, and `urn.ugcPost_urn`. Up to three different numbers, one post.
- **One actor result per Calendar row.** If a share-form ID matches two results, it is a repost and its original. Take the one whose `activity_urn` matches, else the `regular` one. Never add them together.
- **Never pass `fields` to Apify `get-dataset-items`.** It strips nested `stats` and returns zeros without raising an error.
- **Report the match count.** "Matched N of M" every run. If N is less than M, name the misses.
- **Never fabricate a metric.** Public counts come from the actor; private numbers come from the user. If a source is unavailable, say so and leave the field blank.
- **A zero is suspect.** If every count in a run returns 0, treat it as a failed pull, not a real result. Check the `fields` projection first.
- **An empty field is `pending`, not zero.** A number nobody has pulled yet is unknown. Never write 0 to stand for "not collected."
- **Same-sync principle as mark-published:** the Calendar and the metrics log update together, never one without the other.
- **No client names in the content system.** Any engagement listed in `firewall.walled_engagements` never enters it, on either backend.
- Confirm before writing.

Storage discipline is not restated here. **append to a log** and **read the metrics log** in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md` own the format, the idempotency key, and the firewall check on both backends. Read the operation; do not reconstruct it from memory.
