---
name: polish-and-score
description: Polish a draft to sound human and in Brian's voice, then score it. Use when Brian says "polish and score", "polish this", "run the polish chain", "humanize this", or "score this post". Runs two different chains by asset type — Substack newsletters get a newsletter-voice pass and no scoring; LinkedIn posts get a structure + word humanizer pass and a post-score against Brian's real LinkedIn data. Also runs automatically at the end of draft-content.
---

# Polish and score

The quality gate before publishing. A draft is not done until it sounds like Brian wrote it — not like AI wrote it in Brian's general direction — and, for LinkedIn, until it scores against what actually performs on his feed. Two chains, because a newsletter and a LinkedIn post fail in different ways and get measured differently.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before polishing."

Required keys: `voice.sources` (or `voice.style_guide_path`).
Used if present: `voice.newsletter_voice_path`, `content.polish.structure_humanizer_skill` (default `stay-human`), `content.polish.word_humanizer_skill` (default `surface-humanizer`), `content.polish.newsletter_voice_skill` (default `newsletter-voice`), `content.polish.linkedin_scorer_skill` (default `post-scorer`).

## Detect the asset type

- Frontmatter `target: substack`, a `content.newsletter_drafts_path` location, or "issue #N" → **newsletter chain**.
- Frontmatter `target: linkedin`, a `content.linkedin_drafts_path` location, or a short post → **LinkedIn chain**.
- If unclear, ask.

## Newsletter chain (Substack) — no scorer

The scorer's model is LinkedIn engagement, not email, so newsletters are not scored. Run, in order:

1. **Voice pass.** Align to `voice.sources` (`about-me.md` + `voice.md`): operator not marketer, blunt, short declaratives, opens in the problem, ends on the work. Metrics over adjectives.
2. **Word-level humanizer.** Strip surface AI tells. If a `word_humanizer_skill` (default `surface-humanizer`) is installed, invoke it; **otherwise do the pass inline against `anti-ai-writing.md`** — kill the flagged words, cadences, and constructions that rulebook names (no "not X, but Y", no summary transitions, no inflated verbs, no em dashes). Keep going either way; note which path you used.
3. **newsletter-voice.** Apply the newsletter register from `voice.newsletter_voice_path` (`newsletter-voice.md`). If that file doesn't exist yet, run the `newsletter_voice_skill` (`newsletter-voice`) once to create it first, then apply it. **Substack issues only** — never run this on a LinkedIn post.
4. **anti-ai-writing audit.** Final gate: read the draft against `anti-ai-writing.md` and fix anything that fails. Report what changed.

## LinkedIn chain — polish then score

Run, in order:

1. **Voice pass.** Same voice sources; Brian's published LinkedIn style — short lines, specific tool names, closing question.
2. **Structure humanizer (`stay-human`).** Fix structural AI tells — narrative shape, predictable arcs — not just words. Invoke the `structure_humanizer_skill` (default `stay-human`).
3. **Word-level humanizer.** Same as newsletter step 2 (the `surface-humanizer` skill if installed, else inline against `anti-ai-writing.md`).
4. **anti-ai-writing audit.** Final read against the rulebook; fix and report.
5. **Score (`post-scorer`).** Run the `linkedin_scorer_skill` (default `post-scorer`) — it grades the draft against Brian's real LinkedIn performance data (Apify pull or cached). Report the score, what it liked, and the specific changes it drove. If Apify/data isn't available, say so and skip scoring rather than inventing a number.

## Output

For each draft: the revised text, a short list of the changes each stage made, and — for LinkedIn — the score with its reasoning. Brian accepts or pushes back. Never present a scored number you didn't actually get from the scorer.

## Hard rules

- **Right chain for the asset.** newsletter-voice is Substack-only; post-scorer is LinkedIn-only.
- **Never fabricate a score.** If the scorer can't run, say so.
- **Voice is Brian's, not generic.** The hashtag line in `voice.md` reflects an old sample set — follow current practice (hashtags on LinkedIn posts) unless Brian says otherwise.
- **Audit against `anti-ai-writing.md`** is the non-negotiable final step of both chains.
- Sanitize: no client names or confidential detail; OFP never enters the vault.
