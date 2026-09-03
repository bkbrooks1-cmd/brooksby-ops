---
name: mark-published
description: Sync every content tracker the moment a piece goes live. Use when the user says "mark [X] published", "mark issue #N published", "mark [post] posted", "[X] published, url: …", or "log this as published". One confirmation updates the Notion Content Calendar and Content Ideas plus the published log and the backlog together, so the trackers never drift.
---

# Mark published

When the user publishes a newsletter issue or a LinkedIn post, one command updates **every tracker** in a single pass: the Notion Content Calendar, the linked Content Ideas row, the published log, and the backlog. Before this skill existed, "mark published" was improvised and the trackers drifted. This is the definition of that step.

On the `notion` backend the published log and the backlog are views over rows this skill is already writing, so the four updates collapse to two. That collapse belongs to the contract, not to this skill — call the operations and let them decide how many records there are.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before marking published."

Required keys: `notion.content_db`, `notion.content_ideas_db`, `content.backend`.
Used if present: `content.series`, `voice.name`, `voice.sources` / `voice.style_guide_path`.

**Storage.** The log and backlog updates go through named operations in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`. This skill names no file path.

## Inputs

- **Piece identifier** — a post name, issue number ("Issue #3"), or theme. Fuzzy is fine.
- **Platform** — LinkedIn / Substack / Both. If not stated, infer from the URL host (linkedin.com → LinkedIn; substack.com → Substack).
- **URL** — the live link.
- **Date** — default today, America/Phoenix, unless the user gives one.
- **Metrics (optional)** — impressions, reactions, etc. Usually blank at publish; the Friday `collect-metrics` step fills them later.

Ask only for what you cannot infer (usually nothing beyond the URL).

## The routine — four trackers, one confirmation

Propose every change first, then write on a single "yes".

### 1. Notion Content Calendar (`collection://{notion.content_db}`)

- **Find the row.** Fuzzy-match on Title. For a newsletter, match "Issue #N", the title, or the Series. Query by title and by any Scheduled/Drafted row on or near the date before deciding it is missing.
- **If it exists:** set `Status = Posted`, `Platform`, `date:Post date:start = date`, `Live URL = url`.
- **Hook and Hashtags.** Fill from the draft's first line / tags when empty. When the field already holds a value, the rule is about *where the new value came from*, not about whether one exists:
  - **Never overwrite a Hook with an inferred or fabricated line.** If all you have is the draft, and the field is already filled, leave it.
  - **Do overwrite when the user supplies the actual published copy.** Users edit at publish time — on the reference instance this happened on three of the first five issues — so a stored Hook is often a draft line that was never published, and the Calendar ends up holding a hook that appears nowhere. Set the Hook to the real first sentence of the published piece and **say so in the report**, showing the old value and the new one.
  - If the published copy is not in hand and the stored Hook looks stale, say so and ask for the first line rather than guessing.
- **If no row exists:** create one (Title, Status = Posted, Platform, Post date, Live URL, Hook/Hashtags from the draft).
- **Idempotent, but not inert.** If the row is already `Posted` with the same `Live URL`, never create a second row. That is the whole of the guard: it stops duplicates, it does not stop corrections. On a re-run, still apply any field the user has now supplied that the row lacks or holds wrongly — most often the Hook, once the user hands over the copy they actually published. If nothing has changed, report "already logged, nothing to update" and move on.
- **Live URL** is a `url` property; set it as `"Live URL"`. LinkedIn URLs may arrive in either `share-<id>` or `activity-<id>` form. Store whichever the user gives; `collect-metrics` matches on the numeric URN against both forms, so no conversion is needed here.

### 2. Notion Content Ideas (`collection://{notion.content_ideas_db}`)

- If the Calendar row has an `Idea source` relation, open that Content Ideas row and set `Status = Promoted`.
- If there is no linked idea, skip silently.

### 3. The published log — **append to a log**

- Append the entry: number, date, series (from `content.series`), title / theme, hook, live link, metrics TBD. The operation owns the format — read the last entry for the next number and match what is already there.
- **Skip the append if the URL is already logged** (idempotent) — but if the Hook was corrected in step 1, correct it in this entry too, in the same pass. A hook fixed in the Calendar and left stale in the log is exactly the drift this skill exists to prevent.
- Leave everything around the entry intact.

### 4. The backlog — **append to a log**

- If the piece is listed under "In review," mark it **PUBLISHED** with the date and link (or remove the row), and update any bundle note that references it.
- If it is not in the backlog, skip.

### 5. Newsletter with derived posts

The established convention is **one numbered log row per platform** — the Substack issue plus each LinkedIn cut. When a newsletter and its posts publish:
- Create/patch a Content Calendar row per platform, or set the issue row `Platform = Both`.
- Add one published-log entry per platform.
- Handle each URL the user gives; if they publish the cuts on different days, mark each as it goes live (the idempotent guard keeps re-runs clean).

### 6. Report

List every change with links: the Calendar row (Posted + Live URL), the Content Ideas row (Promoted), the new log entry number, and the backlog graduation. Name how many records that actually was on this backend. If any tracker could not be updated, say which and why — never report a write you did not make.

## Hard rules

- **Manual publishing only.** This skill records what the user already posted; it never posts anything.
- **Confirm before writing.** Propose every update, write on one yes.
- **Idempotent.** Never create a duplicate Calendar row or log entry; re-running the same command is safe.
- **No client names in the content system.** Never write confidential or engagement detail into the logs. An engagement listed in `firewall.walled_engagements` never appears there, on either backend.
- **Never fabricate a hook or quote** — pull from the draft, from the user's supplied published copy, or leave the field alone.
- **A stale Hook is correctable, in both places.** Supplied published copy beats a stored draft line. When you correct a Hook, correct it in the Content Calendar row *and* in the published-log entry. Report every Hook change with its before and after.
- **Idempotency stops duplicates, not corrections.** A re-run that supplies new information updates the record; a re-run that supplies nothing new writes nothing.
- Any prose written follows the user's voice per `voice.sources` (about-me + voice.md + voice-exemplars); `anti-ai-writing.md` is the audit gate, reached through `voice.style_guide_path`.

Storage discipline is not restated here. **append to a log** and **update front matter** in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md` own the formats, the idempotency keys, and the firewall check on both backends. Read the operation; do not reconstruct it from memory.
