---
name: surface-humanizer
description: Strip word-level and sentence-level AI tells from a draft, using Brian's anti-ai-writing rulebook as the authority. Use when Brian says "humanize this", "strip the AI tells", "run the word pass", "de-slop this", or when polish-and-score reaches its word-level humanizer step. Pairs with stay-human, which fixes narrative structure — this one fixes the words, phrases, punctuation, and rhythm. Never rewrites the argument, only how it reads.
---

# Surface humanizer

The word-level pass. `stay-human` fixes the shape of a piece; this fixes the surface of it — the banned vocabulary, the throat-clearing, the em dashes, the colon reveals, the metronome sentence lengths. The two are different failures and need different passes.

The authority for every rule here is Brian's rulebook, `anti-ai-writing.md`. This skill is the operational form of it: same rules, run as a repeatable checklist instead of depending on the model reading 2,700 words carefully every time.

## Config

Reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

Required: `voice.style_guide_path` (the rulebook) or `voice.sources` containing it.
Used if present: `voice.newsletter_voice_path` — **for its spelling convention only.** Never apply its register, structure, or section rules here; those are Substack-only and `polish-and-score` applies them at its own step. This skill runs in both chains, so pulling newsletter register into a LinkedIn post is a defect.

If the rulebook cannot be found, say so and run the pass from the built-in checklist below rather than stopping. Note which path you used.

## Read the rulebook first

Open `anti-ai-writing.md` before touching the draft. It is the source of truth and it changes. The checklist below is the run order, not a replacement for reading it.

## The pass, in order

Work through all seven. Do not stop at the banned-word table — that is the easy half and the least detectable.

### 1. Banned words

Delete or replace every entry in the rulebook's banned-word table. The recurring offenders in Brian's drafts: leverage (verb), robust, seamless, comprehensive, streamline, unlock, harness, journey, crucial, impactful, ensure, actionable, ecosystem, deep dive, unpack, move the needle. Replace with the plain word the table gives, or rewrite the sentence.

### 2. Banned phrases

Six families, all in the rulebook:
- **Throat-clearing** — "It's important to note that", "In today's X landscape", "Let's dive into". Delete and start with the point.
- **Empty hedges** — "At the end of the day", "When it comes to", "It's clear that". Say it or cut it.
- **Fake-casual honesty markers** — "Honestly,", "I'll be honest", "The truth is", "Real talk". These imply the rest was dishonest. Cut every one.
- **AI enthusiasm** — "game-changer", "Here's the thing:", "Excited to share", "The good news is".
- **Lazy closers** — "In conclusion", "Only time will tell", "Feel free to reach out", a tacked-on "What do you think?".
- **Corporate filler** — "moving forward", "in order to", "prior to", "due to the fact that".

### 3. Banned structures

- **"Not just X — it's Y."** State the thing flat. Brian's voice profile bans this construction independently.
- **Rhetorical question then answer.** "The result? Chaos." Never.
- **Rule of three.** Count the lists. If everything arrives in threes, break the pattern — the number should match what there is to say.
- **Synonym stacking.** "comprehensive, sophisticated, and robust" is padding.
- **Dramatic fragments for false tension.** Fragments are in Brian's voice as punch lines ("The process did."), not as manufactured suspense. Keep the former, cut the latter.
- **Bold lead-in sentence followed by an explanation paragraph.** This is the tic that showed up twice in the 2026-07-23 test. Check for it explicitly on every draft.

### 4. Punctuation

- **Em dashes: zero.** Brian's standing rule, deliberately tighter than the rulebook's one-per-500-words allowance. Zero is the target and it overrides the looser limit. Convert to commas, colons, or periods. Watch for the parenthetical pair, the dramatic reveal, the list intro, and stacked dashes in one sentence.
- **Colons: one per 300 to 400 words.** Kill the dramatic colon ("There was one problem: trust"), the setup colon ("Here's what most people miss:"), and the false-thesis colon ("The takeaway: start small").
- **Bold and italics: once per 500 words** in prose. Emphasis comes from word choice and sentence order.

### 5. Sentence starters

Cut "So,", "Well,", "Now,", "Look,", "Basically,", "Essentially,", "Certainly,", "Interestingly,". Note the exception: Brian's voice opens sentences with And, But, and Then on purpose. Leave those alone.

### 6. Rhythm and tone

- **Break the metronome.** A run of 15-to-20-word sentences reads generated. Brian's default is short declaratives with an occasional long list-loaded sentence. Vary paragraph length too — one-sentence paragraphs are in voice.
- **Kill relentless positivity.** If something has a downside, the draft says so. Unearned optimism reads corporate.
- **Name the actor.** "It was determined that" — by whom? Passive voice hiding agency gets rewritten.
- **Cut leading participial phrases.** "Leveraging advanced analytics, the team improved conversion" is a fingerprint. Restructure.
- **Cut trailing "ensuring" clauses.** Almost always filler.
- **Cut false authority.** "Research shows", "Studies indicate", "Experts agree" with nothing cited. Either name the source or drop the claim.
- **Cut self-narration.** "This means that", "What this tells us is", "which was surprising because".
- **No tacked-on conclusion.** When the last real point lands, the piece is over.

### 7. The four-question filter

Run every surviving sentence through the rulebook's filter:
1. Can this word or phrase go without losing meaning? Cut it.
2. Is this the simplest way to say it? Simplify.
3. Would Brian say this out loud to a colleague? If not, rewrite.
4. Does this add information or just sound impressive? If the latter, cut it.

## Hard rules

- **American English throughout.** Brian writes American English. If any earlier pass in the chain left British spellings behind — optimised, organisation, realise, behaviour, whilst — fix them here. This is a known recurring drift from `stay-human`.
- **Never change the argument.** This pass changes how a draft reads, never what it claims. If a sentence has to go because it is empty, say so rather than inventing a replacement claim.
- **Never invent a number, a client, or an anecdote.** If a sentence needs a specific and there isn't one, flag it for Brian instead of filling the gap.
- **Keep the voice's deliberate breaks.** Sentence fragments, sentences opening with And or But, and one-sentence paragraphs are Brian's, not AI tells. Do not normalize them out.
- **The rulebook wins on anything it covers that this file does not.** Where the two disagree on a *limit*, the stricter one wins: this file's zero-em-dash rule is deliberately tighter than the rulebook's one-per-500-words, and it stands. Where they disagree on a *rule*, the rulebook is right and this file needs updating.
- **No client names.** OFP never appears in any draft or in `D:\Brain`.

## Output

Return the revised draft, then a short change list grouped by pass: words cut, phrases cut, structures rewritten, punctuation fixed, rhythm changes. Name anything you flagged rather than fixed. No score — scoring is `post-scorer`'s job.
