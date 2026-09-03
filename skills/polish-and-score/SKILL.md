---
name: polish-and-score
description: Polish a draft to sound human and in the user's voice, then score it. Use when the user says "polish and score", "polish this", "run the polish chain", "humanize this", or "score this post". Runs two different chains by asset type — Substack newsletters get a newsletter-voice pass and no scoring; LinkedIn posts get a structure + word humanizer pass and a post-score against the user's real LinkedIn data. Also runs automatically at the end of draft-content.
---

# Polish and score

The quality gate before publishing. A draft is not done until it sounds like the user wrote it — not like AI wrote it in their general direction — and, for LinkedIn, until it scores against what actually performs on their feed. Two chains, because a newsletter and a LinkedIn post fail in different ways and get measured differently.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before polishing."

Required keys: `voice.sources` (or `voice.style_guide_path`). Paths in the `voice` block are relative to the project folder holding the config — resolve them against that folder, not the working directory.
Used if present: `voice.register` (`linkedin` | `newsletter` | `none`; tie-breaker only), `voice.newsletter_voice_path`, `content.polish.structure_humanizer_skill` (default `stay-human`), `content.polish.word_humanizer_skill` (default `surface-humanizer`, shipped in this plugin as `brooksby-ops:surface-humanizer`), `content.polish.newsletter_voice_skill` (default `newsletter-voice`), `content.polish.linkedin_scorer_skill` (default `post-scorer`).

## Detect the asset type

- `target: substack`, a newsletter-target location per **list drafts**, or "issue #N" → **newsletter chain**.
- `target: linkedin`, a LinkedIn-target location per **list drafts**, or a short post → **LinkedIn chain**.
- Still unclear: fall back to `voice.register` — `"newsletter"` or `"linkedin"` picks that chain and says it did. On `"none"`, or when the key is absent, **ask**. An explicit `target` always beats the register; the register only breaks a tie.

Where a draft lives is answered by **list drafts** in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`, which holds one procedure per backend. This skill polishes text handed to it and writes no content itself.

The voice files are different: `voice.sources` and `voice.style_guide_path` are read **directly**. **They are not content storage and never route through the contract** — the rulebook is a property of the author, not of the store the draft happens to live in.

## Newsletter chain (Substack) — no scorer

The scorer's model is LinkedIn engagement, not email, so newsletters are not scored. Run, in order:

1. **Voice pass.** Align to `voice.sources` — `about-me.md`, `voice.md`, and `voice-exemplars.md`, the general voice that applies to every asset. (`newsletter-voice.md` is deliberately not in `voice.sources`; it is the Substack register and is applied at step 3 only. **`anti-ai-writing.md` is deliberately not in `voice.sources` either** — it is reached through `voice.style_guide_path` and is the audit gate at step 4, not a drafting source. Loading it in both places pays for it twice.) Operator not marketer, blunt, short declaratives, opens in the problem, ends on the work. Metrics over adjectives.
2. **Word-level humanizer (`surface-humanizer`).** Invoke the `word_humanizer_skill` — shipped in this plugin as `brooksby-ops:surface-humanizer`. It runs the seven-pass word, phrase, punctuation, and rhythm sweep against `anti-ai-writing.md`. If the skill cannot be found (a partial install), do the pass inline against `anti-ai-writing.md` instead and say which path you used. Never skip the pass.
3. **newsletter-voice.** Apply the newsletter register from `voice.newsletter_voice_path` (`newsletter-voice.md`). **Substack issues only** — never run this on a LinkedIn post.

   **If `voice.newsletter_voice_path` is unset or the file is missing, this step is skipped — the chain does not stop.** Say plainly that the newsletter register was not applied and why, finish steps 1, 2, and 4, and hand back a polished draft. A missing register makes the output less tuned; halting makes it nonexistent. Never fail the whole chain on one absent optional file.

   What to do about the missing file depends on whether one ever existed:
   - **A tuned profile existed** (this instance has published issues — check the Content Calendar for `Status = Posted, Platform = Substack`, or the contract's rerun check). **Do not regenerate it.** The file is refined across published issues and carries its publish-edit learnings. Running the `newsletter_voice_skill` would overwrite it with a fresh archetype and nothing would error; the output would just be worse. Report it as an install problem to fix by restoring the file, skip the step, and complete the chain.
   - **No issue has ever published** (a fresh install). Run the `newsletter_voice_skill` (`newsletter-voice`) once to create it, save it to `voice.newsletter_voice_path`, then apply it. Say that you generated it and that it should be tuned after the first few issues.
   - **Cannot tell.** Skip the step, say you could not tell which case you were in, and complete the chain. Write nothing.
4. **anti-ai-writing audit.** Final gate: read the draft against `anti-ai-writing.md` and fix anything that fails. Then run the spelling check below. Report what changed.

## LinkedIn chain — polish then score

Run, in order:

1. **Voice pass.** Same voice sources — `about-me.md`, `voice.md`, `voice-exemplars.md`; `anti-ai-writing.md` arrives at step 4, not here. The user's published LinkedIn style — short lines, specific tool names, closing question.
2. **Structure humanizer (`stay-human`).** Fix structural AI tells — narrative shape, predictable arcs — not just words. Invoke the `structure_humanizer_skill` (default `stay-human`). **`stay-human` is an account-level skill whose own rules specify British English and instruct the model to match source spelling. The user's convention beats it** — run the spelling check below on its output, every time. This is the step the drift comes from.
3. **Word-level humanizer (`surface-humanizer`).** Same as newsletter step 2.
4. **anti-ai-writing audit.** Final read against the rulebook; fix, run the spelling check, and report.
5. **Score (`post-scorer`).** Run the `linkedin_scorer_skill` (default `post-scorer`) — it grades the draft against the user's real LinkedIn performance data (Apify pull or cached). Report the score, what it liked, and the specific changes it drove. If Apify/data isn't available, say so and skip scoring rather than inventing a number.

Note: `post-scorer` hardcodes its Apify actor at step 2 and does not read `content.metrics.apify_linkedin_actor`. If that config key changes, the scorer keeps using its literal until someone edits the skill.

## The spelling check (run after every structure or humanizer pass, both chains)

**The user's spelling convention wins.** It is stated in `voice.sources` and it is a property of the author, not of a document type, so it holds on LinkedIn posts and Substack issues alike. `newsletter-voice.md` states it for newsletters; it is equally true everywhere else. Any invoked skill's own convention loses to it — including `stay-human`, which will drift the draft the other way if left unchecked. This recurs every run, so check every run, in both chains.

`stay-human` drifts British, so when the user's convention is American English — the case the chain is tuned for — sweep for and revert: optimise/optimised, organisation, realise, recognise, analyse, behaviour, favour, colour, labour, centre, defence, licence (as a noun for permission), whilst, amongst, learnt, travelled, programme, and the -isation family generally. Report any that were found. When the convention runs the other way, sweep in the other direction; the discipline is the same.

## Output

For each draft: the revised text, a short list of the changes each stage made, and — for LinkedIn — the score with its reasoning. The user accepts or pushes back. Never present a scored number you didn't actually get from the scorer.

## Hard rules

- **The user's spelling convention holds on every asset.** Any invoked skill's own convention loses to it, `stay-human` included. Check for British spellings after any structure or humanizer pass, in both chains, every run.
- **Never regenerate a `newsletter-voice.md` that has been tuned.** On an instance that has published Substack issues, the file carries publish-edit learnings and a missing file is an install problem to report, not a file to recreate. On a fresh install with nothing published, generating it once is correct. If you cannot tell which case you are in, write nothing.
- **A missing optional voice file skips its step; it never fails the chain.** Say what was skipped and why, then finish the remaining steps and hand back the draft.
- **Right chain for the asset.** newsletter-voice is Substack-only; post-scorer is LinkedIn-only.
- **Never fabricate a score.** If the scorer can't run, say so.
- **Voice is the user's, not generic.** Where `voice.md` reflects an older sample set, follow current practice unless the user says otherwise.
- **Never invent a number, a client, or an anecdote** to fill a gap a pass opened. Flag it for the user instead.
- **Audit against `anti-ai-writing.md`** is the non-negotiable final step of both chains.
- Sanitize: no client names or confidential detail; an engagement listed in `firewall.walled_engagements` never enters the content system on either backend, including in provenance notes.
