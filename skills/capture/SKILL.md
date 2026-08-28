---
name: capture
description: Capture what came in since last time — recent Granola meetings into Notion as minutes and action items, and self-addressed idea emails from Gmail into the content vault and Notion. Use when the user says "capture", "capture meetings", "capture my meetings", "capture my ideas", "pull my meetings", "write up my meetings", "meeting minutes", "sweep my inbox for ideas", or asks to turn recent calls or emailed notes into notes and tasks. Also runs inside daily-checkin and monday-planner.
---

# Capture

Turn what came in since last time into records the rest of the OS can use. Two independent sweeps:

- **Ideas** (step 0) — self-addressed emails carrying a clip or an idea, into the content vault, Notion Content Ideas, and the Agent Ideas DB. Needs Gmail.
- **Meetings** (steps 1 to 7) — Granola meetings into Notion meeting pages with minutes, action items, and status, with the action items rolled into Tasks. Needs Granola.

The sweeps do not depend on each other. Run whichever ones have their connector; say plainly which one was skipped and why. This is the canonical capture routine; `daily-checkin` runs it as phase 1 of every morning run and `monday-planner` runs it too, so both sweeps must be safe to run repeatedly without creating duplicates. It also stands alone — "capture" on its own runs both sweeps and nothing else.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before capturing."

Required keys: `notion.meetings_db`, `notion.tasks_db`, `notion.engagements_db`, `notion.internal_engagements`, `firewall.no_connector_accounts`.

Every meeting page, action item, and follow-up task this skill creates carries an engagement. The rule and the four internal buckets live in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md` — read it before step 4.

Optional: `capture.lookback` (one of `this_week`, `last_week`, `last_30_days`; default `last_week`). Also optional: a `content` block (see config.example.json) — enables idea notes to the content vault (steps 0 and 5c). If absent, skip step 0 and step 5c silently.

Step 0 also reads `capture.idea_capture` when present:

```json
"idea_capture": {
  "processed_label": "Captured",
  "lookback_days": 30,
  "personal_ref_path": "D:/Brain/05-Training and Content for my reference"
}
```

Defaults if the block is absent: label `Captured`, 30 days, and no `ref` destination (a `ref` item routes to `00-Inbox` instead).

## Prerequisites

Each sweep needs its own connector, and neither blocks the other:

- **Gmail** — required for step 0. Without it, skip step 0 and say so.
- **Granola** — required for steps 1 to 7. Without it, skip the meeting sweep and say so.

If neither connector is available, say so plainly and stop. (Both are optional for the OS overall.)

## The routine

### 0. Sweep Gmail for captured ideas

The user emails themselves clips and ideas from their phone. This step turns that mail into notes. Skip silently if there is no `content` block in config; skip with a one-line note if Gmail is unavailable.

The subject grammar, the routing table, the `Why:` body rule, legacy subject forms, and the dedupe rule live in `${CLAUDE_PLUGIN_ROOT}/references/idea-capture-convention.md`. Read it before running this step — do not reconstruct the parsing rules from memory.

**0a. Query.** Gmail `search_threads` for mail the user sent to themselves, excluding anything already processed:

```
to:{email.monitored_addresses} -label:{idea_capture.processed_label} newer_than:{lookback_days}d in:anywhere
```

Run one query per monitored address. The user sends from several accounts into one inbox, so match on recipient, not sender.

**0b. Parse.** For each thread, read the subject against the convention and pull the full body with `get_thread` (`PLAIN_TEXT`). Extract the `Why:` line, the link, and any body text. Body text in the user's own words is the most valuable part of the capture — carry it into the note, condensed but not paraphrased into blandness.

**0c. Dedupe.** Two ways things double up here:

- Same thread seen twice — the `processed_label` catches this.
- Same URL sent twice on different days, which the label cannot catch because both threads are new. Compare URLs across the batch and against existing clips in `{content.vault_root}/01-Clips` before writing. One URL is one clip.

**0d. Route.** Per the convention's table: `newsletter` and `post` to the matching idea folder under `content.ideas_path`, `agent` to `collection://{notion.agent_ideas_db}`, `ref` to `idea_capture.personal_ref_path`, anything unrecognized to `00-Inbox`.

Two judgment calls belong to this step, not to the parser:

- **Merge near-duplicates.** Two emails pushing the same angle from different source links are one idea note with both clips in `sources`, not two notes. Splitting one idea across two notes is how a backlog starts looking fuller than it is.
- **Flag reruns.** Check the proposed idea against `content.published_log_path` before writing. If it is close to something already published, write the note anyway and put the overlap in the note body as a flag. The user decides whether it is a new angle; the skill does not silently drop it.

**0e. Write.** Vault notes follow the output contract at `content.contract_path` — read that file, do not reconstruct it. `sources` and `used-in` are written from both ends in the same pass: a clip created alongside the idea it feeds carries a `used-in` link back, and the idea names the clip in `sources`.

Notes going to `idea_capture.personal_ref_path` are deliberately outside the contract. That folder is a personal reference shelf, not part of the content system — plain markdown, no frontmatter, no hub link, and never a source for content.

**0f. Mirror content ideas to Notion.** For each `newsletter` or `post` idea note, add a row to `collection://{notion.content_ideas_db}` with Status = Backlog, the Platform matching the qualifier, and the vault path named in Notes. The vault note is the source of truth; the Notion row is the backlog view of it. Clips do not get Notion rows.

**0g. Label.** Apply `idea_capture.processed_label` to each thread only after its note is written and confirmed. Labeling before the write is what makes a failed run lose the thought permanently. Create the label if it does not exist.

Include everything proposed in step 0 in the step 7 disposition, and write it on the same confirmation.

### 1. Pull recent meetings

Call Granola `list_meetings` for the window: the caller's window if invoked from another skill, otherwise `capture.lookback` (default `last_week`). This returns id, title, date, and participants per meeting.

### 2. Dedupe against the Meetings DB

For each meeting, check the Notion Meetings data source `collection://{notion.meetings_db}` for an existing page:

- First match on `Granola link` containing the meeting id.
- Else match on title + date.

Already-captured meetings are skipped silently. This is what makes the routine safe to run from capture, the check-in, and the planner without creating duplicates.

When this dedupe fails it fails quietly: the meeting is re-summarized from scratch, so the duplicate carries a **different title for the same call** and no title- or date-based check will ever find it. The audit query that catches it — grouping the Meetings DB on `Granola link` — is check 3 in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md`. Run it if the user suspects duplicates, or after any run where the Granola link was written late.

### 3. Fetch detail for uncaptured meetings

For each uncaptured meeting, call Granola `get_meetings` (by id) for the AI summary, notes, and attendees. Do not pull the full transcript by default — link to it instead (see minutes format). Only call `get_meeting_transcript` if the user explicitly asks for transcript-level detail on a meeting.

### 4. Infer the engagement

Match the meeting to an active row in the Engagements data source `collection://{notion.engagements_db}` using attendees and title. If one clearly matches, link it.

If it is ambiguous or no client matches, ask the user which engagement before linking — offering the four internal buckets alongside the client list. Do not guess, and do not offer "none." An internal meeting is Business Development when a named opportunity sits behind it and Networking when it is relationship maintenance. This choice propagates into three records: the meeting page (step 5), its action-item tasks (step 6), and the prep task linked back in step 5d. A null here defects all three.

### 5. Build the Meeting page (proposed, not yet created)

For each uncaptured meeting, prepare a page for the Meetings data source with:

- **Name** = meeting title
- **Date** = meeting date
- **Attendees** = participant names (text)
- **Granola link** = the meeting's Granola URL (this is also the dedupe key and the path to the full transcript)
- **Engagement** = inferred relation (or as confirmed in step 4) — always set, client row or internal bucket
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

### 5c. Idea note for teachable insights (only if a `content` block exists in config)

If a meeting surfaced an insight worth teaching publicly — a pattern, lesson, or method, not client specifics — also prepare a markdown idea note for the content vault at `content.ideas_path`:

- **Sanitize.** No client names, no figures, no confidential detail. The insight must stand on its own without the engagement behind it.
- **Firewall.** Meetings on engagements matching `firewall.no_connector_accounts` never feed content. Skip them here entirely, even sanitized.
- **Output contract.** Follow the contract at `content.contract_path` exactly: frontmatter (`type`, `created`, `status`, `tags`) plus a folder-qualified hub wikilink. The paste-block lives in that file — read it, do not reconstruct from memory.
- One insight per note; filename slugged from the insight.

The vault owns content; Notion owns tasks. Do not create a Task or Content Ideas row for the note — the Wednesday synthesis session picks it up from the vault. Include proposed idea notes in the disposition list (step 7) and write them on the same confirmation.

### 5d. Link the prep back to the meeting

A prep brief is written before the meeting exists, so nothing has connected the two yet. Capture is where that closes.

Search the Tasks data source `collection://{notion.tasks_db}` for a prep artifact for this same meeting: a task with `Type = Prep` whose name, engagement, or due date lands within a day or two of the meeting. Also check for a standalone prep page in the workspace with a matching title, which is the older shape and should not exist anymore.

If one is found, propose to:

- set its `Meeting` relation to the meeting page from step 5
- set its `Engagement` to the same engagement
- mark it **Done**

and prepend one line to the top of the minutes body, above the Summary:

```
**Prep for this meeting:** [<prep task name>](<prep task url>)
```

That single link is what makes the plan and the record readable side by side months later. The relation is bidirectional, so the meeting page shows the prep and the prep task shows the meeting.

If more than one prep artifact matches, link the newest and name the others in the disposition so the user can delete them. If none is found, skip this step silently and never create a prep task at capture time.

### 6. Extract action items into Tasks (proposed, not yet created)

For each action item, prepare a row for the Tasks data source `collection://{notion.tasks_db}`:

- **Name** = the action, phrased as an outcome
- **Type** = Follow-up (or Deliverable if it is a work product)
- **Source** = Meeting
- **Due date** = if a date was stated or clearly implied; otherwise leave blank
- **Engagement** = same engagement linked to the meeting. Never blank — if the meeting resolved to an internal bucket, its action items inherit that bucket. An action item that is plainly a different kind of work than the meeting (an invoicing follow-through off a networking call, say) routes on its own merits per the routing reference, and the client-invoicing rule applies.
- **Meeting** = relation to the meeting page created in step 5

### 7. Disposition

Show the user everything proposed before writing anything: the swept idea notes and their destinations (step 0), then the meeting pages to create, and under each, the prep link (step 5d), the action-item Tasks, and any idea notes (step 5c). Then:

- Create only on confirmation. Offer "yes to all" or let the user pick per meeting.
- After creating each Meeting page, link its action-item Tasks via the `Meeting` / `Action items` relation, and link the prep task from step 5d.
- Report what was created with links, and name any stale prep artifacts the user should delete.

## Hard rules

- Create nothing without confirmation. Propose, then write on a yes.
- Never create a duplicate Meeting page — always dedupe in step 2 first.
- Never write a null Engagement on a meeting page, an action item, or a prep task. Every one of them resolves to a client row or an internal bucket.
- Drafts only for anything leaving the building; the user sends. Accounts listed in `firewall.no_connector_accounts` are never connected — capture their meeting notes into Notion as normal (Granola is the user's own record), but never send anything to those accounts.
- Idea notes (steps 0 and 5c) are always sanitized and never come from firewalled engagements. When in doubt about whether a detail is client-confidential, leave it out. An emailed idea that names a firewalled account is generalized if it routes to Notion, and dropped if it was headed for the vault.
- Never apply the processed label before the note is written. A labeled thread with no note is a thought lost silently, and nothing downstream will ever catch it.
- Never write frontmatter or a hub link into `idea_capture.personal_ref_path`. That shelf is outside the content system on purpose.
- If Granola is unavailable mid-run, stop the meeting sweep and say so; do not fabricate minutes. Step 0 still runs.
- If Gmail is unavailable, skip step 0 and say so; the meeting sweep still runs.
- Never edit or re-summarize a meeting page that already exists unless the user asks.
- Never leave a prep artifact unlinked when its meeting page exists. Step 5d is not optional when a match is found.
