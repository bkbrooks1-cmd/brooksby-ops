# Engagement routing

Shared rule for every skill that creates a task, a meeting page, or an action
item. Referenced from `daily-checkin`, `monday-planner`, `build-prep`, `capture`,
and `onboarding`. Change it here, not in the skills.

---

## The convention

> **Every task belongs to an engagement.** Client work links to a client row.
> Non-client work links to one of four internal buckets: Marketing Content,
> Business Admin, Business Development, or Networking. Internal buckets carry no
> Status, Rate, or Billing model, which is what keeps them out of revenue and
> client reporting. A task with a null Engagement is a defect, not a state.

The four buckets live in the Engagements data source alongside client rows, and
their page IDs are in config under `notion.internal_engagements`:

| Bucket | Config key |
|---|---|
| Marketing Content | `notion.internal_engagements.marketing_content` |
| Business Admin | `notion.internal_engagements.business_admin` |
| Business Development | `notion.internal_engagements.business_development` |
| Networking | `notion.internal_engagements.networking` |

## Bucket boundaries

Routing is deterministic. Read the boundaries before asking the user.

- **Marketing Content** — LinkedIn posts, newsletter issues, the Wednesday
  engine, metrics logging, cadence seeding. Anything that produces published
  content.
- **Business Admin** — internal ops. Tool and workspace cleanup, system
  maintenance, taxes, subscriptions, bookkeeping with no client attached.
- **Business Development** — prospect and referral follow-ups with a specific
  opportunity attached but no engagement yet. When a prospect converts, its
  tasks move to the new engagement row.
- **Networking** — relationship maintenance with no specific opportunity.
  Coffee, check-ins, intro calls that aren't pitching anything.

## Client invoicing follows the client

Anything traceable to a specific client invoice — raising it, sending it,
chasing it, watching for payment — links to **that client's engagement row**, not
Business Admin. The client rollup is only useful if it shows everything owed on
the engagement, money included.

Business Admin holds the money work with no client behind it: quarterly taxes,
subscriptions, the accounting tool itself.

## How to apply it

1. Try to match a client engagement first, by attendee domain, company name, or
   the task's subject.
2. No client match: route to the bucket whose boundary fits.
3. Plausibly two buckets: **ask**. Show the two candidates and let the user pick.
4. Never write null. If the user is unavailable to answer, hold the task in the
   proposal list rather than creating it unassigned.

Internal buckets never get Status, Rate, Billing model, or Start date written to
them, and `Weekly report` stays unchecked. Those fields are what the Status =
Active filters and revenue reporting key off; filling them in pulls an internal
bucket into client reporting. Each bucket's page body carries a note saying so —
leave it there.

## The verification queries

Three checks. Run all three — each one passes while the others fail.

### 1 and 2 — null engagements

Capture writes an engagement to three record types, so checking Tasks alone
passes while meeting pages sit orphaned. Check both.

Open tasks with no engagement — expect zero:

```sql
SELECT Name, Status, Type FROM "collection://{notion.tasks_db}"
WHERE (Status IS NULL OR Status != 'Done')
  AND (Engagement IS NULL OR Engagement = '' OR Engagement = '[]')
```

Meeting pages with no engagement — expect zero. Meetings have no Status, so
there is no "closed" state to exclude; every row counts:

```sql
SELECT Name, "date:Date:start" FROM "collection://{notion.meetings_db}"
WHERE Engagement IS NULL OR Engagement = '' OR Engagement = '[]'
```

A hit on the second query with a clean first query is the signature of a meeting
captured before the routing rule was live, or a capture run that skipped step 4.

### 3 — duplicate meeting pages

Capture's step 2 dedupes on the Granola meeting id inside `Granola link`. Nothing
in the OS ever checks whether that dedupe actually held. This is that check —
expect zero rows:

```sql
SELECT "Granola link" AS G, COUNT(*) AS copies,
       GROUP_CONCAT(Name, ' | ') AS names
FROM "collection://{notion.meetings_db}"
WHERE G IS NOT NULL AND G != ''
GROUP BY G HAVING copies > 1
```

Group on the link, not the title. A failed dedupe re-summarizes the meeting from
scratch, so the two copies usually carry **different titles for the same call** —
"Weekly Connect - 2026-07-20" and "1:1 Jennifer — video transcription review" were
one meeting. Matching on name or date finds none of them.

Two things to know before deleting anything:

- **Date-scope the query to nothing.** Scan the whole table. On 2026-08-11 a
  7/27–7/28 window found three duplicates from a run that had actually produced
  seven, and missed two older incidents entirely.
- **The empty copy is usually the duplicate, but not always.** The keeper is
  normally the one holding `Action items`. When *both* copies have action items,
  or a prep task's `Meeting` relation points at the second copy, deleting either
  one loses linked records — surface both and let the user decide. Never
  bulk-delete on the emptiness heuristic alone.

## Config missing

If `notion.internal_engagements` is absent or its values are empty, say so and
stop rather than writing null engagements:

"notion.internal_engagements is not configured. Run onboarding to provision the
four internal engagement buckets, or add their page IDs to config."
