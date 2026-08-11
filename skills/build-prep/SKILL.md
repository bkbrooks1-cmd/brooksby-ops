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

Required keys: `notion.meetings_db`, `notion.tasks_db`, `notion.engagements_db`, `notion.internal_engagements`. Used if present: `email.monitored_addresses`, `voice.style_guide_path`, `firewall.no_connector_accounts`.

The prep task this skill creates always carries an engagement. The rule and the four internal buckets live in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md`.

## Step 1 — Identify the meeting

If the user named a meeting or client, resolve it. Otherwise pull the **next upcoming event** from Google Calendar (skip declined events and personal blocks). State which meeting you are prepping and its date, time, and attendees, so the user can redirect before you spend effort.

## Step 2 — Identify the engagement

Match the meeting to a row in the Engagements data source `collection://{notion.engagements_db}` by attendees and title.

If it is ambiguous or matches nothing, ask which engagement — and offer the four internal buckets alongside the client list, because every meeting resolves to one of them. A non-client meeting is Business Development when a named opportunity sits behind it and Networking when it is relationship maintenance. "None" is not an option; a null engagement here propagates into the prep task and, later, into the meeting page.

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

## Step 5 — Save the brief into the prep task

**The prep task is the brief.** Its page body holds the full text. There is no separate prep page and no separate prep database. One object per meeting, which is what lets capture link the plan to the record afterward.

Show the brief in chat first. Then, on confirmation:

1. **Find or create the prep task** in the Tasks data source `collection://{notion.tasks_db}`.
   - Look for an existing open task with `Type = Prep` whose name, engagement, or due date matches this meeting. Reuse it. Do not create a second one.
   - If none exists, create it: **Name** = `Prep: <meeting name> (<Day M/D H:MM>)`, **Type** = Prep, **Source** = Planning, **Status** = To do, **Due date** = the day before the meeting, **Engagement** = the engagement from step 2 — client row or internal bucket, never blank.
2. **Write the brief into that task's page body.** Replace the body, do not append, so a rebuilt brief never stacks on top of a stale one. Open with a dateline: `Built <YYYY-MM-DD>`, plus a reschedule note if the meeting moved.
3. **Link to the meeting page** through the `Meeting` relation if a Meetings page already exists. Usually it does not yet, because the meeting has not happened. Capture creates the page afterward and links back then (capture skill, step 5d).
4. **One brief per meeting.** If more than one prep artifact exists for the same meeting — a second Prep task, or a standalone prep page left behind by an earlier build or a reschedule — keep the newest and name the stale ones so the user can delete them. Never leave two prep documents for one meeting.

**No Word copy by default.** Generate a `.docx` only when the user asks for one to send to somebody. Use the docx skill, and save it to `<project_root>/deliverables/` when the engagement has a `project_root`.

Create nothing without a yes.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Accounts in `firewall.no_connector_accounts` are never connected — prep their meetings from the user's own records, but send nothing to those accounts.
- Never invent status, action items, or commitments. If the context is thin, say so and prep from what exists.
- The brief lives in the prep task body. Never create a standalone prep page.
- Do not create or close tasks beyond the prep task in Step 5, and only on confirmation.
