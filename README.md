# brooksby-ops

The Solopreneur OS, packaged as a Cowork plugin. Config-driven agents for a solo consultant who runs the business out of Notion, Gmail, and Google Calendar.

The skills carry the logic. Your data lives in one config file. To run this for a different person, you swap the config — not the skills.

## What's in here

| Component | What it does |
|---|---|
| `skills/daily-checkin` | Today's meetings with prep status, email triaged into proposed tasks, deliverable check-off. |
| `skills/monday-planner` | Builds the week plan page in Notion, creates prep tasks, drafts the Monday team email. |

Onboarding, plus the capture / content / lead skills, land in later versions.

## Layout

```
brooksby-ops/
  .claude-plugin/
    plugin.json          # plugin manifest
  skills/
    daily-checkin/SKILL.md
    monday-planner/SKILL.md
  config.example.json    # sanitized template — copy and fill in
  .gitignore             # keeps the real config and About Me/ out of the repo
  README.md
```

## Install

1. Add this plugin to Cowork (Settings > Capabilities), or install the packaged `.plugin` file.
2. Copy `config.example.json` to `solo-os-config.json` in your Solopreneur OS project folder.
3. Fill in your own values — Notion database IDs, monitored email addresses, voice source path. (Onboarding will automate this step in a future version.)
4. Connect Gmail, Google Calendar, and Notion in Cowork.
5. Run a daily check-in to confirm everything returns data.

## How config works

Every skill reads `solo-os-config.json` from the Solopreneur OS project root before it runs. If the file is missing, or a required key is absent, the skill stops and points you to onboarding. No instance data — IDs, emails, paths, customer names, rates — is ever hardcoded in a skill.

The real `solo-os-config.json` is git-ignored and never ships. Only the sanitized `config.example.json` is committed.

## Status

Version 0.1.0 — Phase 1 scaffold. Two skills packaged and config-driven. Not yet beta-ready: onboarding skill and the Notion schema template are the next milestones.
