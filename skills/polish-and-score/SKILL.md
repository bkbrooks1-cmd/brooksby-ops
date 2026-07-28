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
Used if present: `voice.newsletter_voice_path`, `content.polish.structure_humanizer_skill` (default `stay-human`), `content.polish.word_humanizer_skill` (default `surface-humanizer`, shipped in this plugin as `brooksby-ops:surface-humanizer`), `content.polish.newsletter_voice_skill` (default `newsletter-voice`), `content.polish.linkedin_scorer_skill` (default `post-scorer`).

## Detect the asset type

- Frontmatter `target: substack`, a `content.newsletter_drafts_path` location, or "issue #N" → **newsletter chain**.
- Frontmatter `target: linkedin`, a `content.linkedin_drafts_path` location, or a short post → **LinkedIn chain**.
- If unclear, ask.

## Newsletter chain (Substack) — no scorer

The scorer's model is LinkedIn engagement, not email, so newsletters are not scored. Run, in order:

1. **Voice pass.** Align to `voice.sources` — `about-me.md`, `voice.md`, and `anti-ai-writing.md`, the general voice that applies to every asset. (`newsletter-voice.md` is deliberately not in `voice.sources`; it is the Substack register and is applied at step 3 only.) Operator not marketer, blunt, short declaratives, opens in the problem, ends on the work. Metrics over adjectives.
2. **Word-level humanizer (`surface-humanizer`).** Invoke the `word_humanizer_skill` — shipped in this plugin as `brooksby-ops:surface-humanizer`. It runs the seven-pass word, phrase, punctuation, and rhythm sweep against `anti-ai-writing.md`. If the skill cannot be found (a partial install), do the pass inline against `anti-ai-writing.md` instead and say which path you used. Never skip the pass.
3. **newsletter-voice.** Apply the newsletter register from `voice.newsletter_voice_path` (`newsletter-voice.md`). **Substack issues only** — never run this on a LinkedIn post.

   If the file is missing, the right move depends on whether one ever existed:
   - **A tuned profile existed** (this instance has published issues — check the Content Calendar for `Status = Posted, Platform = Substack`, or the vault's published log). **Stop and say so.** The file is refined across published issues and carries its publish-edit learnings. Running the `newsletter_voice_skill` would overwrite it with a fresh archetype and nothing would error; the output would just be worse. The fix is to restore the file.
   - **No issue has ever published** (a fresh install). Run the `newsletter_voice_skill` (`newsletter-voice`) once to create it, save it to `voice.newsletter_voice_path`, then apply it. Say that you generated it and that it should be tuned after the first few issues.
   - **Cannot tell.** Ask before writing anything.
4. **anti-ai-writing audit.** Final gate: read the draft against `anti-ai-writing.md` and fix anything that fails. Then run the spelling check below. Report what changed.

## LinkedIn chain — polish then score

Run, in order:

1. **Voice pass.** Same voice sources; Brian's published LinkedIn style — short lines, specific tool names, closing question.
2. **Structure humanizer (`stay-human`).** Fix structural AI tells — narrative shape, predictable arcs — not just words. Invoke the `structure_humanizer_skill` (default `stay-human`). **`stay-human` is an account-level skill whose own rules specify British English. Brian writes American English, so that rule loses here** — run the spelling check below on its output, every time. This is the step the drift comes from.
3. **Word-level humanizer (`surface-humanizer`).** Same as newsletter step 2.
4. **anti-ai-writing audit.** Final read against the rulebook; fix, run the spelling check, and report.
5. **Score (`post-scorer`).** Run the `linkedin_scorer_skill` (default `post-scorer`) — it grades the draft against Brian's real LinkedIn performance data (Apify pull or cached). Report the score, what it liked, and the specific changes it drove. If Apify/data isn't available, say so and skip scoring rather than inventing a number.

Note: `post-scorer` hardcodes its Apify actor at step 2 and does not read `content.metrics.apify_linkedin_actor`. If that config key changes, the scorer keeps using its literal until someone edits the skill.

## The spelling check (run after every structure or humanizer pass, both chains)

**Brian writes American English.** This is a property of the author, not of a document type, so it holds on LinkedIn posts and Substack issues alike. `newsletter-voice.md` states it for newsletters; it is equally true everywhere else. Any invoked skill's spelling convention loses to it — including `stay-human`, which will drift the draft into British spellings if left unchecked. This recurs every run, so check every run, in both chains.

Sweep for and revert: optimise/optimised, organisation, realise, recognise, analyse, behaviour, favour, colour, labour, centre, defence, licence (as a noun for permission), whilst, amongst, learnt, travelled, programme, and the -isation family generally. Report any that were found.

## Output

For each draft: the revised text, a short list of the changes each stage made, and — for LinkedIn — the score with its reasoning. Brian accepts or pushes back. Never present a scored number you didn't actually get from the scorer.

## Hard rules

- **Brian writes American English, on every asset.** Any invoked skill's spelling convention loses to that, `stay-human` included. Check for British spellings after any structure or humanizer pass, in both chains, every run.
- **Never regenerate a `newsletter-voice.md` that has been tuned.** On an instance that has published Substack issues, the file carries publish-edit learnings and a missing file is an install problem to report, not a file to recreate. On a fresh install with nothing published, generating it once is correct. If you cannot tell which case you are in, ask.
- **Right chain for the asset.** newsletter-voice is Substack-only; post-scorer is LinkedIn-only.
- **Never fabricate a score.** If the scorer can't run, say so.
- **Voice is Brian's, not generic.** The hashtag line in `voice.md` reflects an old sample set — follow current practice (hashtags on LinkedIn posts) unless Brian says otherwise.
- **Never invent a number, a client, or an anecdote** to fill a gap a pass opened. Flag it for Brian instead.
- **Audit against `anti-ai-writing.md`** is the non-negotiable final step of both chains.
- Sanitize: no client names or confidential detail; OFP never enters the vault, including in provenance notes.
