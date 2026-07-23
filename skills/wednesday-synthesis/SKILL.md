---
name: wednesday-synthesis
description: Run the weekly content synthesis — read recent clips, ideas, and metrics, and return three ranked newsletter angles with the LinkedIn posts that would derive from each. Use when Brian says "run my synthesis", "run my weekly synthesis", "synthesize my week", "give me content angles", or "what should I write this week". Brian picks the angle; the skill never chooses. Reads and updates the forward theme map.
---

# Wednesday synthesis

The engine of the weekly loop. Read everything captured recently plus what the metrics say lands, and hand Brian three strong newsletter angles to choose from — each with its supporting material and the posts it would spin off. One thinking effort sets up the whole week. **Brian picks. The skill proposes, ranks, and explains, but never picks the angle.**

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before running synthesis."

Required keys: `content.ideas_path`, `content.vault_root`, `content.metrics_log_path`, `content.theme_map_path`.
Used if present: `content.synthesis_lookback_days` (default 14), `notion.content_ideas_db`, `voice.sources`.

## Sources to read

1. **Clips** — everything in `{vault_root}/01-Clips` created or modified in the last `synthesis_lookback_days` (default 14). Each carries a `Why:` line — that is the signal for why it was kept.
2. **Ideas** — everything in `content.ideas_path` (`02-Ideas`, including `LinkedIn-Posts/` and `Newsletter-Topics/`) modified in the window. An idea note is Brian's own angle, in his words.
3. **Metrics log** — `content.metrics_log_path`. What actually landed: which formats and themes drew impressions and engagement.
4. **Theme map** — `content.theme_map_path`. The forward roadmap: what each series needs next, banked angles, and what has already been covered (never pitch a rerun as new).
5. **Published log** (if present near the ideas path) — `published-posts-log.md`, so a proposed angle knows which past post it builds on.

## What synthesis produces

Three newsletter angles. For each:

- **The core argument** — the thesis in a sentence or two, in Brian's register (operator, not marketer).
- **Supporting material** — which specific clips and idea notes feed it (name the notes), and how deep the cluster is.
- **Derived posts** — how 2–3 LinkedIn posts would come off it (e.g. story / framework / contrarian), so Brian sees the week's full yield.
- **Series fit** — which series it belongs to (Outgrown / PM Track / List of Demands / Standalone) and how it fits the theme map's forward slots.

**Ranking**, in this order: (1) what the metrics say lands, (2) depth and strength of the supporting cluster, (3) fit to Brooksby Consulting's positioning and to the forward theme map. State the ranking and the reason for the top pick.

## On Brian's pick

When Brian chooses one:

1. **Mark the idea note** `status: active` and link the supporting cluster (wikilinks to the feeding clips and ideas), under the output contract.
2. **Update the theme map** (`content.theme_map_path`): slot the chosen theme into the current week, carry the two unpicked angles forward as banked angles, and move the chosen theme out of "forward slots" into recent/used.
3. **Mirror to Notion Content Ideas** (`collection://{notion.content_ideas_db}`), if configured: set the chosen idea's row `Status = Drafting` (or `Next`), so the Notion backlog reflects reality. Propose before writing.
4. Hand off: the next step is `draft-content` ("draft issue #N on [the chosen angle]").

## Hard rules

- **Never pick the angle.** Rank and recommend; Brian decides.
- **Never pitch a rerun as new.** Check the theme map's "already covered" list and the published log first.
- **Borrow ideas with attribution, never prose.** If an angle builds on someone else's post, credit them; never absorb their sentences.
- **Sanitize.** No client names, figures, or confidential detail — the vault is public-facing raw material. OFP never feeds content.
- Propose Notion/theme-map writes before making them.

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
