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

## Config missing

If `notion.internal_engagements` is absent or its values are empty, say so and
stop rather than writing null engagements:

"notion.internal_engagements is not configured. Run onboarding to provision the
four internal engagement buckets, or add their page IDs to config."
