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

Draft in Brian's voice using `voice.sources` — `about-me.md` (who he is), `voice.md` (how he sounds: operator not marketer, blunt, short declaratives, opens in the problem, ends on the work), and `anti-ai-writing.md` (the rulebook). Audit every draft against `anti-ai-writing.md` before showing it. No em dashes, no buzzwords, no soft CTAs, no rhetorical-question openers.

## Phase 1 — the newsletter (editable, with side-by-side variants)

Trigger: "draft issue #N on [angle]".

1. **Draft the issue** into `content.newsletter_drafts_path` as `issue-N_<slug>.md`, from the active idea note (marked by synthesis) and its linked clips. Work in the real example Brian supplies (2–3 sentences, a number if he has one). Output contract applies (frontmatter, `target: substack`, hub link).
2. **Iterate on command.** Brian edits like an editor: "tighten," "cut section two," "that claim is wrong," "punch up the open." Apply directly and keep going.
3. **Side-by-side variants.** When Brian asks for another take — "give me a punchier version," "a second take on the open," "a shorter cut" — write a **parallel draft** in the same folder (`issue-N_<slug>_vB.md`, `_vC.md`), not a replacement, so he can compare versions against each other. Iterate on any variant. Keep a one-line note at the top of each variant saying what makes it different.
4. **Finalize one.** Brian picks the winning draft. Mark it the final (frontmatter `status: drafted`), and **archive the losing variants** (`status: archived`) — do not delete them. Only the finalized issue proceeds.

**Gate:** no LinkedIn post is derived until Brian has finalized one newsletter draft. Verify any claim written as Brian's experience before it ships.

## Phase 2 — derive the posts (variable count)

Trigger: "derive N posts" (or "cut the posts"). Only after Phase 1 is finalized.

1. **How many.** Use the number Brian names this week ("derive two posts" → 2). If he doesn't say, use `content.default_post_count` (default 3). Never assume 3 when he asked for fewer.
2. **Which cuts, scaled to N:**
   - **1** — the single strongest cut (or the type Brian names).
   - **2** — two distinct angles, e.g. story + framework.
   - **3** — story / framework / contrarian.
   - Confirm the mix with Brian in one line if it's ambiguous.
3. **Write** each post into `content.linkedin_drafts_path` as its own note (`issue-N_post-story.md`, etc.), standalone, 150–250 words, in Brian's published LinkedIn style: short lines, zero em dashes, specific tool names, a closing question, hashtags at the bottom. Output contract applies (`target: linkedin`). Audit each against `anti-ai-writing.md`.
4. Hand off to `polish-and-score` ("polish and score").

## Hard rules

- **Newsletter first, posts derived.** Never draft posts before the issue is finalized.
- **Variants are additive.** A new variant never overwrites an existing draft; losing variants are archived, not deleted.
- **Respect the post count.** Derive exactly what Brian asked for.
- **Borrow ideas with attribution, never prose.**
- **Audit against `anti-ai-writing.md`** before calling any draft done. Verify experiential claims.
- **Sanitize.** No client names or confidential detail. OFP never enters the vault.

Output contract (D:\Brain). Every note you create or edit in the capture
folders (00-Inbox, 01-Clips, 02-Ideas, 03-Claude-Sessions, 04-Drafts):
- frontmatter: type (clip|idea|draft|session|reference|hub),
  created (YYYY-MM-DD), status (inbox|active|drafted|published|archived), tags
- drafts also carry: target (linkedin|substack); published + url once live
- one upward wikilink to the folder's _index.md, folder-qualified:
  [[04-Drafts/Newsletters/_index|↑ Newsletters]] or [[04-Drafts/LinkedIn-Posts/_index|↑ LinkedIn Posts]] — never a bare [[_index]]
- wikilinks to related notes you're confident about
Never write into _ref-* folders or project junctions. Never touch OFP.
Never leave a note without frontmatter or a hub link.
