---
name: capture
description: Capture recent meetings from Granola into Notion — meeting minutes, action items, and status — and disposition them. Use when the user says "capture", "capture meetings", "capture my meetings", "pull my meetings", "write up my meetings", "meeting minutes", or asks to turn recent calls into notes and tasks. Also runs as a step inside daily-checkin and monday-planner. Requires the Granola connector.
---

# Capture

Turn recent Granola meetings into Notion meeting pages with minutes, action items, and status — and roll the action items into Tasks. This is the canonical meeting-capture routine; daily-checkin and monday-planner run it too, so it must be safe to run repeatedly without creating duplicates.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before capturing."

Required keys: `notion.meetings_db`, `notion.tasks_db`, `notion.engagements_db`, `firewall.no_connector_accounts`.

Optional: `capture.lookback` (one of `this_week`, `last_week`, `last_30_days`; default `last_week`).

## Prerequisite

The **Granola** connector must be enabled. If it is not, say so plainly and stop — capture has nothing to read without it. (Granola is optional for the OS overall, but required for this skill.)

## The routine

### 1. Pull recent meetings

Call Granola `list_meetings` for the window: the caller's window if invoked from another skill, otherwise `capture.lookback` (default `last_week`). This returns id, title, date, and participants per meeting.

### 2. Dedupe against the Meetings DB

For each meeting, check the Notion Meetings data source `collection://{notion.meetings_db}` for an existing page:

- First match on `Granola link` containing the meeting id.
- Else match on title + date.

Already-captured meetings are skipped silently. This is what makes the routine safe to run from capture, the check-in, and the planner without creating duplicates.

### 3. Fetch detail for uncaptured meetings

For each uncaptured meeting, call Granola `get_meetings` (by id) for the AI summary, notes, and attendees. Do not pull the full transcript by default — link to it instead (see minutes format). Only call `get_meeting_transcript` if the user explicitly asks for transcript-level detail on a meeting.

### 4. Infer the engagement

Match the meeting to an active row in the Engagements data source `collection://{notion.engagements_db}` using attendees and title. If one clearly matches, link it. If it is ambiguous or nothing matches, ask the user which engagement (or none) before linking — do not guess.

### 5. Build the Meeting page (proposed, not yet created)

For each uncaptured meeting, prepare a page for the Meetings data source with:

- **Name** = meeting title
- **Date** = meeting date
- **Attendees** = participant names (text)
- **Granola link** = the meeting's Granola URL (this is also the dedupe key and the path to the full transcript)
- **Engagement** = inferred relation (or as confirmed in step 4)
- **Body (minutes)**, in this order:
  - **Summary** — 2 to 4 sentences from the Granola AI summary.
  - **Decisions** — bullets, only if any were made.
  - **Discussion notes** — the substantive points, condensed.
  - **Action items** — bullet per item with owner and due date if stated (these become Tasks in step 6).
  - **Status / next** — where the engagement or topic stands after this meeting, and the next checkpoint.
  - A final line: "Full transcript: <Granola link>".

Write minutes in the user's voice per `voice.style_guide_path` if set; otherwise neutral plain-professional. No buzzwords, no em dashes.

### 5b. Word copy for client-facing meetings

If the meeting is client-facing (it has external attendees) and its engagement has a `project_root` in config, also produce a Word (.docx) copy of the same minutes and save it to that engagement's folder — for example `<project_root>/deliverables/Minutes_<YYYY-MM-DD>_<Client>.docx`. Use the docx skill for clean formatting. The Word file is the shareable artifact; the Notion page stays the system of record. Internal-only meetings, or meetings whose engagement has no `project_root`, get the Notion page only. Include the Word file in the disposition list and create it on the same confirmation.

### 6. Extract action items into Tasks (proposed, not yet created)

For each action item, prepare a row for the Tasks data source `collection://{notion.tasks_db}`:

- **Name** = the action, phrased as an outcome
- **Type** = Follow-up (or Deliverable if it is a work product)
- **Source** = Meeting
- **Due date** = if a date was stated or clearly implied; otherwise leave blank
- **Engagement** = same engagement linked to the meeting
- **Meeting** = relation to the meeting page created in step 5

### 7. Disposition

Show the user everything proposed before writing anything: the meeting pages to create, and under each, the action-item Tasks. Then:

- Create only on confirmation. Offer "yes to all" or let the user pick per meeting.
- After creating each Meeting page, link its action-item Tasks via the `Meeting` / `Action items` relation.
- Report what was created with links.

## Hard rules

- Create nothing without confirmation. Propose, then write on a yes.
- Never create a duplicate Meeting page — always dedupe in step 2 first.
- Drafts only for anything leaving the building; the user sends. Accounts listed in `firewall.no_connector_accounts` are never connected — capture their meeting notes into Notion as normal (Granola is the user's own record), but never send anything to those accounts.
- If Granola is unavailable mid-run, stop and say so; do not fabricate minutes.
- Never edit or re-summarize a meeting page that already exists unless the user asks.
