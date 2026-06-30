# brooksby-ops

The Solopreneur OS, packaged as a Cowork plugin. Config-driven agents for a solo consultant who runs the business out of Notion, Gmail, and Google Calendar.

The skills carry the logic. Your data lives in one config file. To run this for a different person, you swap the config — not the skills.

## What's in here

| Component | What it does |
|---|---|
| `skills/onboarding` | First-time setup. Provisions the six Notion databases, captures their IDs into config, connects the tools, and verifies with a check-in. |
| `skills/daily-checkin` | Today's meetings with prep status, email triaged into proposed tasks, deliverable check-off. |
| `skills/monday-planner` | Builds the week plan page in Notion, creates prep tasks, drafts the Monday team email. |

The capture / content / lead skills land in later versions.

## Layout

```
brooksby-ops/
  .claude-plugin/
    plugin.json                              # plugin manifest
  skills/
    onboarding/
      SKILL.md                               # first-time setup
      references/notion-schema.md            # the six-database schema spec
    daily-checkin/SKILL.md
    monday-planner/SKILL.md
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
5. Say **"set up the OS"** to run onboarding. It creates the six Notion databases, builds your home dashboard, writes the config, and verifies the install with a check-in.

After that, say "check in" for the daily view or "plan my week" for the Monday planner.

## How config works

Every skill reads `solo-os-config.json` from the Solopreneur OS project root before it runs. If the file is missing, or a required key is absent, the skill stops and points you to onboarding. No instance data — IDs, emails, paths, customer names, rates — is ever hardcoded in a skill.

The real `solo-os-config.json` is git-ignored and never ships. Only the sanitized `config.example.json` is committed.

## Status

Version 0.3.1 — Phase 1. Onboarding skill written and fully dry-run verified against the live Notion connector: six databases + two-way relations build cleanly in a fresh workspace, a check-in read works end to end, and the home dashboard (six linked views) builds programmatically. Install narrative rewritten around onboarding, plus a Notion access pre-check and user-confirmed config folder. Three skills packaged and config-driven. Remaining before beta: align the daily skills' config-path lookup with the user-chosen folder, then run an unattended full-flow test with a real tester.
