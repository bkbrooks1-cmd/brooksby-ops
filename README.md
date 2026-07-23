# brooksby-ops

The Solopreneur OS, packaged as a Cowork plugin. Config-driven agents for a solo consultant who runs the business out of Notion, Gmail, and Google Calendar.

The skills carry the logic. Your data lives in one config file. To run this for a different person, you swap the config — not the skills.

## What's in here

| Component | What it does |
|---|---|
| `skills/onboarding` | First-time setup. Provisions the seven Notion databases, captures their IDs into config, connects the tools, and verifies with a check-in. |
| `skills/daily-checkin` | Today's meetings with prep status, email triaged into proposed tasks, deliverable check-off, plus recent meetings to capture. |
| `skills/monday-planner` | Builds the week plan page in Notion, creates prep tasks, captures last week's meetings, drafts the Monday weekly email (generic by default; per-engagement templates via config). With a content vault configured, the plan also carries the standing content deliverables with their draft paths. |
| `skills/capture` | Turns recent Granola meetings into Notion meeting pages with minutes, action items, and status — plus a Word copy to the engagement folder for client-facing meetings, and (with a content vault configured) a sanitized idea note when a meeting surfaces a teachable insight. Runs standalone ("capture") and as a step inside the check-in and planner. Requires Granola. |
| `skills/friday-wrap` | Closes out the week: confirms what got done, rolls overdue open work forward as carryover, and writes a week-wrap page so Monday starts honest. With a content vault configured, unpublished drafts roll as carryover and the vault-health checklist runs with the wrap. Say "weekly wrap". |
| `skills/build-prep` | On-demand prep brief for a named or next meeting: last-meeting status, open items owed, recent email, and agenda in one place. Say "build the prep". |

The dedicated content-engine and lead-capture skills land in later versions. In the meantime, v0.10.0 adds optional **content vault hooks**: point the config at a local markdown vault (Obsidian or plain folders) and the planner, capture, and wrap pick up content work alongside client work. Division of labor: Notion owns tasks and calendar; the vault owns drafts and research.

## Layout

```
brooksby-ops/
  .claude-plugin/
    plugin.json                              # plugin manifest
  skills/
    onboarding/
      SKILL.md                               # first-time setup
      references/notion-schema.md            # the seven-database schema spec
    daily-checkin/SKILL.md
    capture/SKILL.md                         # Granola meetings -> Notion minutes + tasks (+ Word copy)
    friday-wrap/SKILL.md                     # weekly close-out + carryover
    build-prep/SKILL.md                      # on-demand meeting prep brief
    monday-planner/
      SKILL.md
      references/weekly-email.md             # generic vs per-engagement email setup
  templates/
    weekly_email_TEMPLATE.html               # generic weekly email, copy per engagement
  config.example.json                        # sanitized template (onboarding writes the real one)
  .gitignore                                 # keeps the real config and About Me/ out of the repo
  README.md
```

## Install

Onboarding does the setup for you — you do not hand-fill database IDs.

1. Add this plugin to Cowork (Settings > Capabilities).
2. Connect Notion, Gmail, and Google Calendar in Cowork.
3. In Notion, share the page you'll use as the OS home with the Notion integration, so the plugin can create databases under it.
4. Pick the Cowork project folder where the OS will live — this is where your `solo-os-config.json` gets written.
5. Say **"set up the OS"** to run onboarding. It creates the seven Notion databases, builds your home dashboard, writes the config, and verifies the install with a check-in.

After that, say "check in" for the daily view or "plan my week" for the Monday planner.

## How config works

Every skill finds `solo-os-config.json` by searching your connected Cowork project folder(s) by filename before it runs — no fixed path is assumed, so it works whatever you named the folder. If the file is missing, or a required key is absent, the skill stops and points you to onboarding. No instance data — IDs, emails, paths, customer names, rates — is ever hardcoded in a skill.

The real `solo-os-config.json` is git-ignored and never ships. Only the sanitized `config.example.json` is committed.

One optional block worth knowing about: `content` (shape in `config.example.json` under `_content_example`). Add it to hook the OS into a local content vault — the planner lists the standing content deliverables, capture writes sanitized idea notes under the vault's output contract, and the wrap rolls unpublished drafts and runs the vault-health checklist. Leave it out and the OS runs exactly as before. Two rules always hold: idea notes are sanitized (no client names, figures, or confidential detail), and engagements in `firewall.no_connector_accounts` never feed content.

## Status

Version 0.10.0 — Phase 1 plus capture, Friday wrap, on-demand meeting prep, and a Content Ideas backlog; the weekly loop (Monday plan → prep → capture → Friday wrap) is complete to the architecture spec. Onboarding fully dry-run verified against the live Notion connector: the databases + two-way relations build cleanly in a fresh workspace, a check-in read works end to end, and the home dashboard (linked views) builds programmatically. Install rewritten around onboarding, with a Notion access pre-check and user-confirmed config folder. All skills locate config by filename across connected folders; descriptions are person-neutral. The Monday planner's weekly email is a generic out-of-the-box template with per-engagement custom templates driven by config. The `capture` skill turns Granola meetings into Notion minutes, action items, and status — run standalone, or as a step inside the check-in and planner (dedupes against the Meetings DB, so it is safe to run from any entry point). v0.8.0 adds a seventh database, Content Ideas — a backlog of LinkedIn/Substack article ideas with monetization angles, linked to the Content Calendar. v0.9.0 turns the content pair into a full LinkedIn/Substack workflow: Content Ideas gains Hook and Theme; Content Calendar becomes the post pipeline **and** published log (Platform, Hook, Hashtags, Series/Series week, lightweight Impressions/Reactions analytics) with the draft/final copy in each post's page body. v0.10.0 adds the optional content vault hooks: a `content` config block wires the planner (standing deliverables: Newsletter #N, LinkedIn posts x3, Metrics log), capture (sanitized idea notes under the vault's output contract; firewalled engagements never feed content), and the Friday wrap (unpublished drafts roll as carryover; vault-health checklist) into a local markdown vault. Remaining before beta: one unattended full-flow test with a real tester. Still unbuilt by design (post-MVP): the full content engine and lead/idea capture.
