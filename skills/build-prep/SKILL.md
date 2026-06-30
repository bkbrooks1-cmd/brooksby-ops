---
name: build-prep
description: Build a prep document for an upcoming meeting — named or the next one on the calendar. Use when the user says "build the prep", "build prep", "prep for [meeting/client]", "prep me for my next meeting", "meeting prep", or asks to get ready for a meeting. Pulls prior-meeting status, open action items, recent email, and the engagement record into one brief.
---

# Build prep

Produce a focused prep brief for one meeting so the user walks in ready. Gather the context that is already in the system — last meeting, open items, recent email, the engagement record — and turn it into a short brief. Save it only if the user wants it kept.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before building prep."

Required keys: `notion.meetings_db`, `notion.tasks_db`, `notion.engagements_db`. Used if present: `email.monitored_addresses`, `voice.style_guide_path`, `firewall.no_connector_accounts`.

## Step 1 — Identify the meeting

If the user named a meeting or client, resolve it. Otherwise pull the **next upcoming event** from Google Calendar (skip declined events and personal blocks). State which meeting you are prepping and its date, time, and attendees, so the user can redirect before you spend effort.

## Step 2 — Identify the engagement

Match the meeting to a row in the Engagements data source `collection://{notion.engagements_db}` by attendees and title. If it is ambiguous or matches nothing, ask which engagement (or none — some meetings are not engagement work).

## Step 3 — Gather context

Pull only what is already in the system. Name any source that fails; never invent context.

1. **Last meeting(s)** with this engagement — most recent pages in the Meetings data source `collection://{notion.meetings_db}`: their status/next notes and action items. If Granola is connected, `query_granola_meetings` can fill detail on what was last discussed.
2. **Open tasks** for the engagement — Tasks data source `collection://{notion.tasks_db}`, Status not Done, especially overdue items and anything Type = Prep or Follow-up tied to this engagement or meeting.
3. **Recent email** — if Gmail is connected, the last ~14 days of threads with the meeting's attendees (addresses in `email.monitored_addresses` all forward in): unresolved asks, commitments made, open questions.
4. **Engagement record** — status, billing model, key contacts, and the `Weekly report` flag.

## Step 4 — Build the prep brief

Assemble a short brief, in this order:

1. **Meeting** — date, time, attendees, and the meeting's purpose in one line.
2. **Where things stand** — engagement status plus the "status / next" from the last meeting.
3. **Open items owed** — action items from the last meeting and their status, split into "on me" and "on them."
4. **What to cover** — proposed agenda and any decisions that need to be made.
5. **Questions to raise** — the unresolved asks pulled from email and open tasks.
6. **Watch items** — risks, overdue commitments, anything likely to come up.

Keep it tight — a brief, not a dossier. Write in the user's voice per `voice.style_guide_path` if set; otherwise neutral plain-professional. No buzzwords, no em dashes.

## Step 5 — Show, then offer to save

Show the brief in chat first. Then offer to keep it, on confirmation:

- **Notion** — a prep page under the meeting's Meetings page (or the home page if none exists yet).
- **Word copy** — for a client-facing meeting whose engagement has a `project_root` in config, a `.docx` in `<project_root>/deliverables/` (use the docx skill).
- **Mark prep done** — if a Prep task exists for this meeting, offer to mark it Done.

Create nothing without a yes.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Accounts in `firewall.no_connector_accounts` are never connected — prep their meetings from the user's own records, but send nothing to those accounts.
- Never invent status, action items, or commitments. If the context is thin, say so and prep from what exists.
- Do not create or close tasks beyond the optional prep-task mark in Step 5, and only on confirmation.
