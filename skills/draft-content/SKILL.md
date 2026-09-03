---
name: draft-content
description: Draft the weekly newsletter issue and derive LinkedIn posts from it, in the user's voice. Use when the user says "draft issue #N", "draft the newsletter", "draft issue #N on [angle]", then "derive N posts", "cut the posts", or "derive the LinkedIn posts". Newsletter first, editable with side-by-side variants; posts derived only after the issue is finalized, and the number of posts is whatever the user asks for that week (default from config).
---

# Draft content

Turn the chosen synthesis angle into a finished newsletter issue, then cut LinkedIn posts from it. Two phases with a hard gate between them: **the newsletter is finalized before any post exists.** One thinking effort, several assets — but the newsletter carries the thinking and the posts derive from it, never the reverse.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before drafting."

Required keys: `content.backend`, `voice.sources` (or `voice.style_guide_path`).
Used if present: `content.default_post_count` (default 3), `voice.name`, `voice.newsletter_voice_path`.

**Storage.** This skill names no path and no database. Every read and write goes through a named operation in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md`, which holds one procedure per backend.

## Voice

Draft in the user's voice using `voice.sources` — `about-me.md` (who they are), `voice.md` (how they sound), and `voice-exemplars.md` (five complete clean pieces; read all five, never the closest in topic). **`anti-ai-writing.md` is not a drafting source** — it is reached through `voice.style_guide_path` as the audit gate. Draft against `voice.md`'s `<avoid>` bank, then audit every draft against `anti-ai-writing.md` before showing it. No em dashes, no buzzwords, no soft CTAs, no rhetorical-question openers. American English throughout.

## Naming (match what is already stored, not a scheme)

There is an established convention. Follow it exactly; a new asset that sorts or reads differently from the ones already stored is a defect. These are titles — **write a note** turns a title into a file or a row, depending on the backend.

| Asset | Title |
|---|---|
| Newsletter issue | `Issue 0N - <Title>` (zero-padded through Issue 09) |
| Newsletter variant | `Issue 0N - <Title> (<variant descriptor>)` |
| LinkedIn post | `Post - <Title>` |
| Standalone brief | `Brief - <Title>` — a newsletter-target asset |
| Standalone framework | `Framework - <Title>` — a newsletter-target asset |

Title is sentence case, spelled out, no slug. Variant descriptor says what makes it different in two or three words — `(newsletter-voice cut)`, `(humanized v2)`, `(punchier open)` — not `_vB`.

Get the issue number from **list drafts**, which counts published issues as well as open ones. Reusing a number that is already published is the failure this exists to prevent.

## Phase 1 — the newsletter (editable, with side-by-side variants)

Trigger: "draft issue #N on [angle]".

1. **Draft the issue** through **write a note**, kind `draft`, target substack, titled `Issue 0N - <Title>`, from the active idea (marked by synthesis) and its linked clips. Work in the real example the user supplies (2–3 sentences, a number if they have one). The operation owns the born-complete rules — including the provenance pair below, which is not optional.
2. **Iterate on command.** The user edits like an editor: "tighten," "cut section two," "that claim is wrong," "punch up the open." Apply directly and keep going.
3. **Side-by-side variants.** When the user asks for another take — "give me a punchier version," "a second take on the open," "a shorter cut" — write a **parallel draft** titled `Issue 0N - <Title> (<variant descriptor>)`, not a replacement, so the versions can be compared against each other. Iterate on any variant. Keep a one-line note at the top of each variant saying what makes it different.
4. **Finalize one.** The user picks the winning draft. Mark it the final (`status: drafted`) and **archive the losing variants** (`status: archived`) through **update front matter** — do not delete them. Only the finalized issue proceeds.

**Gate:** no LinkedIn post is derived until the user has finalized one newsletter draft. Verify any claim written as the user's own experience before it ships.

## Provenance — links run both directions

A clip that fed a draft has to be reachable from the draft, and the draft has to be reachable from the clip. One direction alone rots: a store can accumulate dozens of clips with zero backlinks when nothing ever writes the return link.

**Every asset this skill creates carries a `sources` list** — links to the clips and ideas it drew from. Inherit it from the idea's `sources`, written by `wednesday-synthesis`. If the idea has no `sources` key, reconstruct the list from the clips actually used and say in one line that you did, so the gap in the handoff is visible. An asset genuinely written from the user's head with no feeding clip carries `sources: []`, never a missing key.

**Then stamp the return link on each source clip** through **update front matter**: add the new draft to that clip's `used-in`, creating the key if absent and appending if present. Never overwrite an existing entry, never remove one, and change nothing else about the clip — not its body, not its tags, not its status. That append is the only write this skill ever makes to a clip.

Do this at the end of Phase 1 for the finalized issue only, not for variants, and again at the end of Phase 2 for each derived post. Losing variants get archived without stamping anything.

Report the stamps in one line: how many clips were updated and which. If a clip named in `sources` cannot be resolved, say which one and keep going — a broken source reference is worth surfacing, not worth stopping the draft over.

## Phase 2 — derive the posts (variable count)

Trigger: "derive N posts" (or "cut the posts"). Only after Phase 1 is finalized.

1. **How many.** Use the number the user names this week ("derive two posts" → 2). If they don't say, use `content.default_post_count` (default 3). Never assume 3 when they asked for fewer.
2. **Which cuts, scaled to N:**
   - **1** — the single strongest cut (or the type the user names).
   - **2** — two distinct angles, e.g. story + framework.
   - **3** — story / framework / contrarian.
   - Confirm the mix in one line if it's ambiguous.
3. **Write** each post through **write a note**, kind `draft`, target linkedin, titled `Post - <Title>` — standalone, 150–250 words, in the user's published LinkedIn style: short lines, zero em dashes, specific tool names, a closing question, hashtags at the bottom. Add a `derived/issue-0N` tag and a link back to the source issue so the post is not an orphan. Carry the issue's `sources` onto the post and stamp `used-in` on each source clip, per the provenance section above. Audit each against `anti-ai-writing.md`.
4. Hand off to `polish-and-score` ("polish and score").

## Hard rules

- **Newsletter first, posts derived.** Never draft posts before the issue is finalized.
- **Match the stored naming.** `Issue 0N - <Title>` and `Post - <Title>`. No slugs, no `_vB` suffixes.
- **Every derived post links back to its source issue.** A post with no outbound link is an orphan.
- **Provenance is written, not remembered.** Every asset carries `sources`; every source clip gets `used-in` stamped back. A draft that shipped without both is incomplete, even if the clips were named in chat.
- **Writing to a clip is limited to the `used-in` key.** Append only. Never edit a clip's body, tags, or status.
- **Variants are additive.** A new variant never overwrites an existing draft; losing variants are archived, not deleted.
- **Respect the post count.** Derive exactly what the user asked for.
- **Borrow ideas with attribution, never prose.**
- **Audit against `anti-ai-writing.md`** before calling any draft done. Verify experiential claims.
- **Sanitize.** No client names or confidential detail. An engagement listed in `firewall.walled_engagements` never enters the content system on either backend — including in provenance or sourcing notes, and including in the text of a rule describing the wall. Documenting the exception is not an exception.

Storage discipline is not restated here. **write a note**, **list drafts**, and **update front matter** in `${CLAUDE_PLUGIN_ROOT}/references/content-storage.md` own the destinations, the born-complete rules, the hub-link and naming mechanics per backend, and the firewall check. Read the operation; do not reconstruct it from memory.
