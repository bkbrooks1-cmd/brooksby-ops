---
name: draft-content
description: Draft the weekly newsletter issue and derive LinkedIn posts from it, in Brian's voice. Use when Brian says "draft issue #N", "draft the newsletter", "draft issue #N on [angle]", then "derive N posts", "cut the posts", or "derive the LinkedIn posts". Newsletter first, editable with side-by-side variants; posts derived only after the issue is finalized, and the number of posts is whatever Brian asks for that week (default from config).
---

# Draft content

Turn the chosen synthesis angle into a finished newsletter issue, then cut LinkedIn posts from it. Two phases with a hard gate between them: **the newsletter is finalized before any post exists.** One thinking effort, several assets — but the newsletter carries the thinking and the posts derive from it, never the reverse.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

If the file is missing, or any required key below is absent, stop and say:
"solo-os-config.json not found (or missing key: <key>). Run onboarding to set up this OS before drafting."

Required keys: `content.newsletter_drafts_path`, `content.linkedin_drafts_path`, `content.ideas_path`, `voice.sources` (or `voice.style_guide_path`).
Used if present: `content.default_post_count` (default 3), `voice.newsletter_voice_path`.

## Voice

Draft in Brian's voice using `voice.sources` — `about-me.md` (who he is), `voice.md` (how he sounds: operator not marketer, blunt, short declaratives, opens in the problem, ends on the work), and `voice-exemplars.md` (five complete clean pieces; read all five, never the closest in topic). **`anti-ai-writing.md` is not a drafting source** — it is reached through `voice.style_guide_path` as the audit gate. Draft against `voice.md`'s `<avoid>` bank, then audit every draft against `anti-ai-writing.md` before showing it. No em dashes, no buzzwords, no soft CTAs, no rhetorical-question openers. American English throughout.

## File naming and hub links (match the vault, not a scheme)

The vault has an established convention. Follow it exactly; a new file that sorts or reads differently from the ones already on disk is a defect.

| Asset | Filename |
|---|---|
| Newsletter issue | `Issue 0N - <Title>.md` (zero-padded through Issue 09) |
| Newsletter variant | `Issue 0N - <Title> (<variant descriptor>).md` |
| LinkedIn post | `Post - <Title>.md` |
| Standalone brief | `Brief - <Title>.md` — lands in `content.newsletter_drafts_path` |
| Standalone framework | `Framework - <Title>.md` — lands in `content.newsletter_drafts_path` |

Title is sentence case, spelled out, no slug. Variant descriptor says what makes it different in two or three words — `(newsletter-voice cut)`, `(humanized v2)`, `(punchier open)` — not `_vB`.

**Hub link: `[[04-Drafts/_index|↑ Drafts]]`, on every draft in both subfolders.** There is no `04-Drafts/Newsletters/_index.md` and no `04-Drafts/LinkedIn-Posts/_index.md`, and there should not be. `04-Drafts/_index.md` is the hub for both subfolders and says so in its own text. Never write a link to a subfolder index; it will not resolve.

## Phase 1 — the newsletter (editable, with side-by-side variants)

Trigger: "draft issue #N on [angle]".

1. **Draft the issue** into `content.newsletter_drafts_path` as `Issue 0N - <Title>.md`, from the active idea note (marked by synthesis) and its linked clips. Work in the real example Brian supplies (2–3 sentences, a number if he has one). Output contract applies (frontmatter, `target: substack`, hub link) — including the provenance contract below, which is not optional.
2. **Iterate on command.** Brian edits like an editor: "tighten," "cut section two," "that claim is wrong," "punch up the open." Apply directly and keep going.
3. **Side-by-side variants.** When Brian asks for another take — "give me a punchier version," "a second take on the open," "a shorter cut" — write a **parallel draft** in the same folder as `Issue 0N - <Title> (<variant descriptor>).md`, not a replacement, so he can compare versions against each other. Iterate on any variant. Keep a one-line note at the top of each variant saying what makes it different.
4. **Finalize one.** Brian picks the winning draft. Mark it the final (frontmatter `status: drafted`), and **archive the losing variants** (`status: archived`) — do not delete them. Only the finalized issue proceeds.

**Gate:** no LinkedIn post is derived until Brian has finalized one newsletter draft. Verify any claim written as Brian's experience before it ships.

## Provenance — links run both directions

A clip that fed a draft has to be reachable from the draft, and the draft has to be reachable from the clip. One direction alone rots: the vault currently holds 43 clips with zero backlinks because nothing ever wrote the return link.

**Every asset this skill creates carries a `sources:` list in frontmatter** — folder-qualified wikilinks to the clips and idea notes it drew from. Inherit it from the idea note's `sources:` key, written by `wednesday-synthesis`. If the idea note has no `sources:` key, reconstruct the list from the clips actually used and say in one line that you did, so the gap in the handoff is visible. An asset genuinely written from Brian's head with no feeding clip carries `sources: []`, never a missing key.

**Then stamp the return link on each source clip.** For every clip in that `sources:` list, add the new draft to the clip's own frontmatter under `used-in:` as a folder-qualified wikilink, creating the key if it is absent and appending if it already exists. Never overwrite an existing `used-in:` entry, never remove one, and change nothing else in the clip — not its body, not its tags, not its status. This is the only circumstance in which this skill writes into `01-Clips`.

Do this at the end of Phase 1 for the finalized issue only, not for variants, and again at the end of Phase 2 for each derived post. Losing variants get archived without stamping anything.

Report the stamps in one line: how many clips were updated and which. If a clip named in `sources:` cannot be found on disk, say which one and keep going — a broken source reference is worth surfacing, not worth stopping the draft over.

## Phase 2 — derive the posts (variable count)

Trigger: "derive N posts" (or "cut the posts"). Only after Phase 1 is finalized.

1. **How many.** Use the number Brian names this week ("derive two posts" → 2). If he doesn't say, use `content.default_post_count` (default 3). Never assume 3 when he asked for fewer.
2. **Which cuts, scaled to N:**
   - **1** — the single strongest cut (or the type Brian names).
   - **2** — two distinct angles, e.g. story + framework.
   - **3** — story / framework / contrarian.
   - Confirm the mix with Brian in one line if it's ambiguous.
3. **Write** each post into `content.linkedin_drafts_path` as its own note, named `Post - <Title>.md`, standalone, 150–250 words, in Brian's published LinkedIn style: short lines, zero em dashes, specific tool names, a closing question, hashtags at the bottom. Output contract applies (`target: linkedin`, `[[04-Drafts/_index|↑ Drafts]]`). Add a `derived/issue-0N` tag and a wikilink back to the source issue so the post is not an orphan. Carry the issue's `sources:` list onto the post and stamp `used-in:` on each source clip, per the provenance section above. Audit each against `anti-ai-writing.md`.
4. Hand off to `polish-and-score` ("polish and score").

## Hard rules

- **Newsletter first, posts derived.** Never draft posts before the issue is finalized.
- **Match the vault's naming.** `Issue 0N - <Title>.md` and `Post - <Title>.md`. No slugs, no `_vB` suffixes.
- **Hub link is `[[04-Drafts/_index|↑ Drafts]]`.** The subfolder indexes do not exist. Do not link to them and do not create them.
- **Every derived post links back to its source issue.** A post with no outbound wikilink is an orphan under the output contract.
- **Provenance is written, not remembered.** Every asset carries `sources:`; every source clip gets `used-in:` stamped back. A draft that shipped without both is incomplete, even if the clips were named in chat.
- **Writing into `01-Clips` is limited to the `used-in:` key.** Append only. Never edit a clip's body, tags, or status.
- **Variants are additive.** A new variant never overwrites an existing draft; losing variants are archived, not deleted.
- **Respect the post count.** Derive exactly what Brian asked for.
- **Borrow ideas with attribution, never prose.**
- **Audit against `anti-ai-writing.md`** before calling any draft done. Verify experiential claims.
- **Sanitize.** No client names or confidential detail. An engagement listed in `firewall.walled_engagements` never enters the vault — including in provenance or sourcing notes, and including in the text of a rule describing the wall. Documenting the exception is not an exception.

Output contract (D:\Brain). Every note you create or edit in the capture
folders (00-Inbox, 01-Clips, 02-Ideas, 03-Claude-Sessions, 04-Drafts):
- frontmatter: type (clip|idea|draft|session|reference|hub),
  created (YYYY-MM-DD), status (inbox|active|drafted|published|archived), tags
- drafts also carry: target (linkedin|substack); published + url once live
- one upward wikilink to the folder's _index.md, folder-qualified:
  [[04-Drafts/_index|↑ Drafts]] for everything under 04-Drafts, both
  subfolders included — never a bare [[_index]], never a subfolder index
- wikilinks to related notes you're confident about
Never write into _ref-* folders or project junctions. Never touch a walled
engagement (`firewall.walled_engagements`).
Never leave a note without frontmatter or a hub link.
