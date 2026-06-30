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

Required keys: `notion.home_page`, `notion.tasks_db`. Used if present: `notion.leads_db`, `notion.meetings_db`.

## Sources to read

1. **Notion Tasks** (`collection://{notion.tasks_db}`): tasks marked Done with a due date or completion in the last 7 days (what got done), and all open tasks (Status not Done) with their due dates (what is still live, and what is overdue).
2. **Notion Leads** (`collection://{notion.leads_db}`), if present: stage changes and any lead with a next-action date that has passed.
3. **Notion Meetings** (`collection://{notion.meetings_db}`), if present: meetings captured this week, for the record.

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

## Output: the week-wrap page in Notion

Create a sub-page of the home page (page_id `{notion.home_page}`), titled "Week wrap — YYYY-MM-DD" (this Friday's date), with sections in order:

1. **Completed this week** — done items by engagement.
2. **Carryover** — what rolled forward, with the new dates the user confirmed.
3. **Still open and on track** — open tasks due next week, untouched.
4. **Pipeline / meetings** — the one-liners from step 3, if available.
5. **Seed for Monday** — a short note the Monday planner can start from: the carried-forward priorities and anything explicitly flagged.

Write in the user's voice per `voice.style_guide_path` if set; otherwise neutral plain-professional. No buzzwords, no em dashes.

## Hard rules

- Never close a task or move a due date without explicit confirmation. Propose, then apply on a yes.
- Done items are left Done; "archived" here means they simply fall out of the active views, not deleted.
- Drafts only for anything leaving the building; the user sends. Accounts in `firewall.no_connector_accounts` are never connected.
- If a source fails, name it at the top of the wrap and continue with what is available.
