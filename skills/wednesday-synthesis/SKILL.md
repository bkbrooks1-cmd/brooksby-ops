---
name: wednesday-synthesis
description: Run the weekly content synthesis — read recent clips, ideas, and metrics, and return three ranked newsletter angles with the LinkedIn posts that would derive from each. Use when the user says "run my synthesis", "run my weekly synthesis", "synthesize my week", "give me content angles", or "what should I write this week". The user picks the angle; the skill never chooses. Reads and updates the forward theme map.
---

# Wednesday synthesis

The engine of the weekly loop. Read everything captured recently plus what the metrics say lands, and hand the user three strong newsletter angles to choose from — each with its supporting material and the posts it would spin off. One thinking effort sets up the whole week. **The user picks. The skill proposes, ranks, and explains, but never picks the angle.**

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before running synthesis."

Required keys: `content.backend`, `content.series`.
Used if present: `content.synthesis_lookback_days` (default 14), `voice.name`, `voice.sources`.

**Storage.** This skill names no path and no database. Every read and write goes through a named operation in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`, which holds one procedure per backend and says which config keys each backend needs.

## Sources to read

1. **Clips** — **read a clip** for everything captured or modified in the last `synthesis_lookback_days` (default 14). Each carries a `Why` line — that is the signal for why it was kept.
2. **Ideas** — every idea in the same window, newsletter and LinkedIn alike. An idea note is the user's own angle, in their words.
3. **Metrics** — **read the metrics log**. What actually landed: which formats and themes drew impressions and engagement.
4. **Theme map** — the forward roadmap: what each series needs next, banked angles, and what has already been covered (never pitch a rerun as new). The contract names where it lives on each backend.
5. **Already published** — the same operation's rerun check, so a proposed angle knows which past post it builds on and never arrives as a repeat.

## What synthesis produces

Three newsletter angles. For each:

- **The core argument** — the thesis in a sentence or two, in the user's register per `voice.sources` (operator, not marketer).
- **Supporting material** — which specific clips and idea notes feed it, each named as a resolvable link to the stored item, not as a bare title or a prose description. This list is the angle's provenance and it carries forward into the draft, so it has to be link-shaped from the moment it is proposed. Also say how deep the cluster is.
- **Derived posts** — how 2–3 LinkedIn posts would come off it (e.g. story / framework / contrarian), so the user sees the week's full yield.
- **Series fit** — which series it belongs to, chosen from `content.series`, and how it fits the theme map's forward slots. Never invent a series that is not on that list.

**Ranking**, in this order: (1) what the metrics say lands, (2) depth and strength of the supporting cluster, (3) fit to the user's positioning per `voice.sources` and to the forward theme map. State the ranking and the reason for the top pick.

## On the user's pick

When the user chooses one:

1. **Mark the idea** `status: active` through **update front matter**. The operation carries the status onto the backlog view where one exists, so the pipeline and the backlog never disagree.
2. **Record the provenance so `draft-content` inherits it.** Write the chosen angle's supporting cluster into the idea's `sources` — the same links from "Supporting material" above — through the same operation. This is the handoff: `draft-content` reads `sources` and carries it into the draft. An angle handed off without a `sources` list is an incomplete handoff, even when the cluster was named in chat. If an angle genuinely came from the user's head with no feeding clip, write `sources: []` rather than omitting the key, so "no source" is distinguishable from "never recorded."
3. **Update the theme map**: slot the chosen theme into the current week, carry the two unpicked angles forward as banked angles, and move the chosen theme out of "forward slots" into recent/used.
4. Hand off: the next step is `draft-content` ("draft issue #N on [the chosen angle]").

## Hard rules

- **Never pick the angle.** Rank and recommend; the user decides.
- **Never pitch a rerun as new.** Check the theme map's "already covered" list and the rerun check first.
- **Never name a source without linking it.** Every clip or idea cited as supporting material is a resolvable link, in the proposal and in `sources`. Provenance that only exists in the chat transcript is lost the moment the session ends — that is how a content store ends up holding dozens of clips and zero backlinks.
- **Borrow ideas with attribution, never prose.** If an angle builds on someone else's post, credit them; never absorb their sentences.
- **Sanitize.** No client names, figures, or confidential detail — everything here is public-facing raw material. An engagement listed in `firewall.walled_engagements` never enters the content system, on either backend.
- Propose every write before making it.

Storage discipline is not restated here. **write a note** and **update front matter** in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md` own the born-complete rules, the provenance pair, and the firewall check on both backends. Read the operation; do not reconstruct it from memory.
