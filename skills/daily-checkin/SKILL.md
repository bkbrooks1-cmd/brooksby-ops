---
name: daily-checkin
description: The full daily run — captures new meetings and emailed ideas, shows today's meetings with prep status, triages email into tasks, checks off deliverables, updates the plan, and refreshes the Daybook. Use whenever the user says "check in", "what's on my plate", "what does today look like", "morning rundown", "catch me up", or asks for a status of their day or tasks at any point. If the user asks what they should be doing right now, use this skill.
---

# Daily check-in

The two-minute view that keeps the plan honest between Mondays. Read fast, show what matters, update only what the user confirms.

One command, three phases, in this order:

1. **Capture** — pull in what arrived since last time, so the view is built on current data
2. **The today view** — meetings, tasks, email, deliverables
3. **The Daybook** — refresh the snapshot, last, so it reflects everything the first two phases just wrote

The order is the whole design. The Daybook is a read layer; rendering it before capture means it shows a picture that was already stale when it was drawn.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path. (Onboarding writes this file to the folder the user chose at setup.)

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before running the check-in."

Required keys: `notion.tasks_db`, `notion.leads_db`, `notion.agent_ideas_db`, `notion.meetings_db`, `notion.engagements_db`, `notion.internal_engagements`, `email.monitored_addresses`, `firewall.no_connector_accounts`.

Optional `checkin` block controls which phases run:

```json
"checkin": {
  "run_capture": true,
  "render_daybook": true
}
```

Both default to `true` when the block or a key is absent — the full run is the intended shape. Set either to `false` to get the today view alone.

## Engagement routing

Every task this skill proposes carries an engagement. Never propose or create a task with a null Engagement. The rule, the four internal buckets, and how to pick between them live in `${CLAUDE_PLUGIN_ROOT}/references/engagement-routing.md` — read it before proposing tasks.

## Phase 1 — Capture

Skip if `checkin.run_capture` is `false`.

Run the `capture` skill in full: its Gmail idea sweep (step 0) and its Granola meeting sweep (steps 1 to 7). The canonical steps live there; do not restate or reimplement them here. Capture is safe to run repeatedly — both sweeps dedupe — which is what makes it safe to fold into a routine that runs every morning.

Present capture's disposition and the today view's proposals in **one confirmation**, not two. The user should approve their morning once.

Both sweeps are optional on their connectors. If Gmail is down the idea sweep skips; if Granola is down the meeting sweep skips; if both are down phase 1 reports that and the check-in continues to phase 2. A missing connector shrinks the run, it never stops it.

**Where email splits.** Both this phase and the today view read Gmail, and they must not both act on the same message:

- Mail matching the idea-capture convention (`Clip`/`Idea` plus a qualifier, per `${CLAUDE_PLUGIN_ROOT}/references/idea-capture-convention.md`) belongs to capture step 0. It becomes a note, never a task.
- Everything else — real correspondence, requests, questions — belongs to the today view's **From email** block. It becomes a task, never a note.

An emailed idea that also becomes a task is the failure mode to watch for. Capture's processed label runs first, so by the time the today view reads Gmail those threads are already labeled: exclude `-label:{capture.idea_capture.processed_label}` from the today view's query.

## Phase 2 — the today view

### Sources to read

Flag any source that fails; never silently omit it.

0. **Connector health** — run the probes in the render-daybook skill's `references/connector-health.md` first. Failures print as the first line of the today view, in the three-fact form that file specifies. Healthy connectors get no line. A connector that returns an empty result without throwing is the failure this catches.

1. **Google Calendar** (connector): today and tomorrow, all calendars.
2. **Gmail** (connector): since the last check-in (default: last 24 hours). Unread or flagged threads and direct asks, including mail to any address in `email.monitored_addresses` (all forward here). Exclude threads carrying the capture processed label — phase 1 already handled those.
3. **Notion Tasks**: data source `collection://{notion.tasks_db}`. Due today, overdue, and In progress. Read this *after* phase 1, so tasks capture just created appear in today's view rather than surfacing tomorrow.

### Output: the today view, in chat

Keep it tight. Four blocks, skip any that are empty:

1. **Today's meetings** — time, title, prep status. Tomorrow's meetings that need prep today get one line each.
2. **Due and overdue** — tasks due today, then overdue, each with engagement. Overdue items get a suggested move: do today, reschedule (propose a date), or flag for Friday wrap.
3. **From email** — new items that look like work. For each, propose a task (name, due date, engagement). The engagement is not optional: match a client row first, and when nothing matches, route to an internal bucket per the routing reference. Where two buckets both fit, show both and ask. Create rows only after the user confirms. Mark Source = Email.
4. **Deliverables** — anything produced in the current session that maps to an open task; offer to mark it Done.
5. **Captured** — one line summarizing what phase 1 brought in: meetings written, ideas routed, and anything it flagged. Not a re-listing; the disposition already showed the detail. Skip the block when capture found nothing or did not run.

## Phase 3 — refresh the Daybook

Skip if `checkin.render_daybook` is `false`.

Run the `render-daybook` skill. It renders last on purpose: it is a read layer over the same databases phases 1 and 2 just wrote to, so rendering it earlier produces a snapshot that is stale before the user sees it.

Render only after the user has confirmed the writes from phases 1 and 2. A Daybook drawn from proposals the user then declines shows work that does not exist.

`render-daybook` updates the existing artifact in place via `daybook.artifact_id`. It must never create a second one — a new id strands the user's pinned entry. If the render fails, say so in one line and end the check-in normally; the today view is the deliverable, the Daybook is the bonus.

## Update rules

- Create tasks only with confirmation. Batch the proposals so the user can say "yes to all" or pick. Every proposal shows its engagement; a proposal without one is not ready to show.
- Never move a due date or close a task Claude did not create without an explicit yes.
- "Posted", "done", "sent" from the user about a named item = mark the matching task Done without re-asking.
- New leads spotted in email: offer a row in Leads (`collection://{notion.leads_db}`) with Stage = Lead, Source = Email inquiry, and a drafted next action.
- Agent ideas the user voices mid-check-in go to Agent Ideas (`collection://{notion.agent_ideas_db}`) immediately; low risk, no confirmation needed, show the row.

## Hard rules

- Drafts only for anything leaving the building; the user sends. Accounts listed in `firewall.no_connector_accounts` are never connected.
- A failed source means a smaller view, not a silent one: name the connector, how long it has been silent, and the fix, then continue. Canonical wording in `references/connector-health.md`.
- **Phase order is fixed: capture, then the today view, then the Daybook.** Each phase reads what the one before it wrote.
- Never reimplement capture or the Daybook here. Call those skills. Two copies of a routine drift, and the copy inside another skill is the one nobody remembers to update.
- Never let one phase's failure end the run. A dead connector in phase 1 still leaves a today view worth reading.
- One confirmation gate for phases 1 and 2 together. Approving a morning should not take three rounds.
- The user can still run `capture` or `render-daybook` on their own. Folding them into the check-in adds a caller; it does not take away the standalone command.
