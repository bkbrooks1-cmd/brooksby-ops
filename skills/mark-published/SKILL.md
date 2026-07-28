---
name: mark-published
description: Sync every content tracker the moment a piece goes live. Use when the user says "mark [X] published", "mark issue #N published", "mark [post] posted", "[X] published, url: …", or "log this as published". One confirmation updates the Notion Content Calendar and Content Ideas plus the vault published-posts-log and content-backlog together, so the trackers never drift.
---

# Mark published

When Brian publishes a newsletter issue or a LinkedIn post, one command updates **all four trackers** in a single pass: the Notion Content Calendar, the linked Content Ideas row, the vault `published-posts-log.md`, and the vault `content-backlog.md`. Before this skill existed, "mark published" was improvised and the trackers drifted. This is the definition of that step.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before marking published."

Required keys: `notion.content_db`, `notion.content_ideas_db`, `content.published_log_path`, `content.backlog_path`.
Used if present: `content.ideas_path`, `voice.sources` / `voice.style_guide_path`.

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
  - **Do overwrite when Brian supplies the actual published copy.** He edits at publish time — he did on 7/28, and on Issues #2 and #3 — so a stored Hook is often a draft line that was never published, and the Calendar ends up holding a hook that appears nowhere. Set the Hook to the real first sentence of the published piece and **say so in the report**, showing the old value and the new one.
  - If the published copy is not in hand and the stored Hook looks stale, say so and ask for the first line rather than guessing.
- **If no row exists:** create one (Title, Status = Posted, Platform, Post date, Live URL, Hook/Hashtags from the draft).
- **Idempotent, but not inert.** If the row is already `Posted` with the same `Live URL`, never create a second row. That is the whole of the guard: it stops duplicates, it does not stop corrections. On a re-run, still apply any field the user has now supplied that the row lacks or holds wrongly — most often the Hook, once Brian hands over the copy he actually published. If nothing has changed, report "already logged, nothing to update" and move on.
- **Live URL** is a `url` property; set it as `"Live URL"`. LinkedIn URLs may arrive in either `share-<id>` or `activity-<id>` form. Store whichever Brian gives; `collect-metrics` matches on the numeric URN against both forms, so no conversion is needed here.

### 2. Notion Content Ideas (`collection://{notion.content_ideas_db}`)

- If the Calendar row has an `Idea source` relation, open that Content Ideas row and set `Status = Promoted`.
- If there is no linked idea, skip silently.

### 3. Vault `published-posts-log.md` (`content.published_log_path`)

- Append the next-numbered row to the table:
  `| N | date | Series | Title / theme | Hook | [post](url) — metrics TBD |`
  (Match the file's existing column order exactly; read the last row to get the next number and the format.)
- **Skip the append if the URL is already in the log** (idempotent) — but if the Hook was corrected in step 1, correct it in this row too, in the same pass. A hook fixed in Notion and left stale in the vault is exactly the drift this skill exists to prevent.
- Keep the file's frontmatter and trailing sections (Themes covered, Recycle pool, hub link) intact.

### 4. Vault `content-backlog.md` (`content.backlog_path`)

- If the piece is listed under "In review," mark it **PUBLISHED** with the date and link (or remove the row), and update any bundle note that references it.
- If it is not in the backlog, skip.

### 5. Newsletter with derived posts

The established convention is **one numbered log row per platform** — the Substack issue plus each LinkedIn cut. When a newsletter and its posts publish:
- Create/patch a Content Calendar row per platform, or set the issue row `Platform = Both`.
- Add one `published-posts-log.md` row per platform.
- Handle each URL the user gives; if they publish the cuts on different days, mark each as it goes live (the idempotent guard keeps re-runs clean).

### 6. Report

List every change with links: the Calendar row (Posted + Live URL), the Content Ideas row (Promoted), the new log row number, and the backlog graduation. If any tracker could not be updated, say which and why — never report a write you did not make.

## Hard rules

- **Manual publishing only.** This skill records what Brian already posted; it never posts anything.
- **Confirm before writing Notion.** Propose all four updates, write on one yes.
- **Idempotent.** Never create a duplicate Calendar row or log line; re-running the same command is safe.
- **No client names in the vault.** Never write confidential or engagement detail into the logs. OFP never appears in `D:\Brain`.
- **Never fabricate a hook or quote** — pull from the draft, from Brian's supplied published copy, or leave the field alone.
- **A stale Hook is correctable, in both places.** Supplied published copy beats a stored draft line. When you correct a Hook, correct it in the Content Calendar row *and* in the `published-posts-log.md` row. Report every Hook change with its before and after.
- **Idempotency stops duplicates, not corrections.** A re-run that supplies new information updates the record; a re-run that supplies nothing new writes nothing.
- Any prose written follows Brian's voice per `voice.sources` (about-me + voice.md + anti-ai-writing).

Output contract (D:\Brain). Every note you create or edit in the capture
folders (00-Inbox, 01-Clips, 02-Ideas, 03-Claude-Sessions, 04-Drafts):
- frontmatter: type (clip|idea|draft|session|reference|hub),
  created (YYYY-MM-DD), status (inbox|active|drafted|published|archived), tags
- drafts also carry: target (linkedin|substack); published + url once live
- one upward wikilink to the folder's _index.md, folder-qualified:
  [[02-Ideas/_index|↑ Ideas]] — never a bare [[_index]]
- wikilinks to related notes you're confident about
Never write into _ref-* folders or project junctions. Never touch OFP.
Never leave a note without frontmatter or a hub link.
