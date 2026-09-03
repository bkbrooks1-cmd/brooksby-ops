---
name: friday-wrap
description: Close out the week so Monday starts from an honest picture. Use when the user says "weekly wrap", "wrap", "friday wrap", "close out the week", "wrap up my week", or "end of week". Reviews what got done, rolls overdue open work forward as carryover, and writes a week-wrap page in Notion.
---

# Friday wrap

The Friday close-out. Confirm what got done, roll the open work forward so the Monday planner does not start from a stale picture, and leave a short record. Read fast, change only what the user confirms.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before the weekly wrap."

Required keys: `notion.week_plans_page`, `notion.tasks_db`. Used if present: `notion.leads_db`, `notion.meetings_db`, `notion.content_db`, and a `content` block (see config.example.json) — enables the content-drafts check, the metrics step, and the hygiene step below.

**Storage.** The content steps name no path. They call named operations in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`, which holds one procedure per backend.

`notion.week_plans_page` is where the finished wrap lands. It is the same page the Monday planner writes week plans to, so plans and wraps sit together in date order. If that key is absent, fall back to `notion.home_page` and say so in the output.

## Sources to read

1. **Notion Tasks** (`collection://{notion.tasks_db}`): tasks marked Done with a due date or completion in the last 7 days (what got done), and all open tasks (Status not Done) with their due dates (what is still live, and what is overdue).
2. **Notion Leads** (`collection://{notion.leads_db}`), if present: stage changes and any lead with a next-action date that has passed.
3. **Notion Meetings** (`collection://{notion.meetings_db}`), if present: meetings captured this week, for the record.
4. **Open content drafts**, only if a `content` block exists in config: **list drafts** — everything still at `status: drafted`, newsletter and LinkedIn alike.

## What the wrap does

### 1. Completed this week

List the Done items from the last 7 days, grouped by engagement. This is the honest record of the week's output. Leave them Done — they already drop out of the active dashboard views; nothing to change.

### 2. Carryover — roll open work forward

For every open task whose due date is before next Monday (overdue or due over the weekend), propose a disposition so Monday starts clean:

- **Move to next week** — propose a specific new due date.
- **Keep as-is** — leave it, it is genuinely due now.
- **Drop** — close it if it no longer matters.

Show the list with a proposed move per item. Apply only on confirmation — never move a due date or close a task without an explicit yes. Batch it so the user can say "move all to Monday" or pick.

### 3. Pipeline and meetings (only if those DBs are present)

One line each: leads whose next action is past due (flag for Monday), and the count of meetings captured this week. Keep it short — this is a record, not a report.

### 4. Content drafts and hygiene (only if a `content` block exists in config)

Unpublished drafts roll forward, same as tasks: everything **list drafts** returns at `status: drafted` goes into Carryover with a link to where it lives, so the Monday planner picks it up. Anything published this week gets counted in Completed. Never change a draft's status here — publishing is manual; this step only reads.

Then run **the content hygiene check** and report the result as one-liners: what passed, what needs a fix. The contract says what the check is on this backend — a checklist on one, three queries on the other — and carries the clip-prune caution, which is the one part of this step that can lose material. Propose fixes; apply only on confirmation.

### 5. Content metrics (only if a `content` block exists in config)

Run the metrics collection for the pieces published this week — the canonical steps live in the `collect-metrics` skill. In short: find this week's `Status = Posted` rows in the Content Calendar, auto-pull the public LinkedIn counts (reactions/comments/reposts) if an Apify actor is configured, and prompt the user for the private numbers (LinkedIn impressions + profile views, Substack opens/open-rate/new-subs). Write the results to the matching Content Calendar rows **and** the metrics log through **append to a log** — both together, never one without the other. Report what was written and name any post whose 7-day window hasn't closed yet so it carries to next week. Confirm before writing.

## Output: the week-wrap page in Notion

Create a sub-page of the Weekly Plans and wraps page (page_id `{notion.week_plans_page}`), titled "Week wrap — YYYY-MM-DD" (this Friday's date). Never create it under `notion.home_page` — the wrap belongs alongside the week plans, not at the root. Sections in order:

1. **Completed this week** — done items by engagement.
2. **Carryover** — what rolled forward, with the new dates the user confirmed. Includes unpublished content drafts, each linked to where it lives, when the content block is configured.
3. **Still open and on track** — open tasks due next week, untouched.
4. **Pipeline / meetings** — the one-liners from step 3, if available.
5. **Content hygiene** — the one-liners from step 4, if the content block is configured.
6. **Content metrics** — the numbers logged this week and anything still pending, if the content block is configured.
7. **Seed for Monday** — a short note the Monday planner can start from: the carried-forward priorities and anything explicitly flagged.

Write in the user's voice per `voice.style_guide_path` if set; otherwise neutral plain-professional. No buzzwords, no em dashes.

## Hard rules

- Never close a task or move a due date without explicit confirmation. Propose, then apply on a yes.
- Done items are left Done; "archived" here means they simply fall out of the active views, not deleted.
- Drafts only for anything leaving the building; the user sends. Accounts in `firewall.no_connector_accounts` are never connected.
- If a source fails, name it at the top of the wrap and continue with what is available.
