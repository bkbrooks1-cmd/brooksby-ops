# Idea capture convention — email to vault

The rule for turning a self-addressed email into a note. One grammar, read by
the Gmail sweep in `capture` step 0.

## Subject format

```
<Verb> <qualifier>: <title>
```

Two tokens, a colon, then a human-readable title. The parser reads the first
two words; the title is for the person.

## Verb — provenance

| Verb | Means | Note type |
|---|---|---|
| `Clip` | Someone else's material being kept — a link, article, or post | `clip` |
| `Idea` | The user's own angle, in their words | `idea` |

## Qualifier — destination

| Qualifier | Destination | Config key |
|---|---|---|
| `newsletter` | Newsletter topic idea | `content.ideas_path`/Newsletter-Topics |
| `post` | LinkedIn post idea | `content.ideas_path`/LinkedIn-Posts |
| `agent` | Notion Agent Ideas DB — never the vault | `notion.agent_ideas_db` |
| `ref` | Personal reference shelf — outside the content system | `content.personal_ref_path` |

A `Clip` with a content qualifier lands in `01-Clips` and the qualifier records
which idea folder it feeds. An `Idea` with a content qualifier lands in the
matching idea folder.

## Body

First line is `Why:` followed by one sentence on why it was kept. That line is
the signal `wednesday-synthesis` reads, so it survives into the note verbatim.
The link goes below it. Everything after is free text and gets condensed into
the note.

If no `Why:` line is present, infer one from the subject and body and mark the
note `status: inbox` so it surfaces for review.

## Examples

```
Clip post: every process step has to earn its spot
Idea newsletter: the Solopreneur OS as a working companion
Idea agent: company process brain for SOP access
Clip ref: Marsham workout and food metrics
```

## Unrecognized subjects

A missing or unknown qualifier is not an error. Route to `00-Inbox` with
`status: inbox` and name it in the disposition. The convention must never lose
a thought because the user was typing on a phone.

## Legacy subjects

Mail sent before this convention used loose forms: `Post idea:`,
`LinkedIn post idea-`, `Idea -`, `Clip-`, `Agent idea-`. The sweep matches these
too, mapping `post idea`/`linkedin post idea` → `post`, `newsletter idea` → `newsletter`,
`agent idea` → `agent`, and a bare `Idea`/`Clip` → unqualified (00-Inbox).
Do not remove legacy matching — the user's phone habits predate the rule.

## Dedupe

The `Captured` Gmail label is the processed marker. The sweep queries
`-label:Captured` and applies the label only after the note is written and
confirmed. A thread already carrying the label is skipped silently, the same
way `capture` step 2 dedupes meetings on the Granola link.

Two threads carrying the same URL are one clip, not two. The user forwards
things twice. Match on the URL in the body before writing.

## Firewall

Accounts in `firewall.no_connector_accounts` never feed content. If a captured
idea names one, route it to Notion if it is an `agent` idea and generalize the
client reference, or drop it if it was headed for the vault. The vault rule is
absolute: no client names in the content vault, sanitized or not.
