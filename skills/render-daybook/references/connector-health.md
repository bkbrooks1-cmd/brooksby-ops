# Connector health — the shared probe

Canonical steps. `render-daybook` runs these before it renders. `daily-checkin` runs them before it prints the today view. Both surfaces get the same answer, because a connector that dies silently on one surface and loudly on the other is worse than one that dies quietly on both.

This exists because a real tester ran for roughly six weeks with a dead Gmail connector and the OS never said a word.

## The probes

Read `daybook.connectors` from config for the list to run. Anything in `daybook.optional_connectors` is optional: silence from it is not a failure.

| Connector | Probe | Fail signal |
|---|---|---|
| Notion | query the Tasks data source | error, or an empty schema |
| Google Calendar | list events for today | error, or zero events on a weekday when the last ten weekdays averaged more than two |
| Gmail | search threads, last 24 hours | error, or zero threads in 48 hours |
| QuickBooks | get company info | error or auth failure |
| Granola | list meetings, last 7 days | error only — optional connector, silence is not a failure |

Run every configured probe even if an earlier one failed. One dead connector should not hide a second.

**A connector whose tools are absent has failed.** Treat a missing tool exactly like a thrown error, not like "not configured." When a connector is disconnected the whole server disappears and no call is made at all, so nothing errors and nothing looks wrong — which is precisely how a connector stays dead for six weeks. If a name in `daybook.connectors` has no tools available this session, that is a failure. Verified live on 2026-08-09 with Gmail.

Skip a probe when the thing it feeds is switched off. If `daybook.revenue.source` is `"none"`, QuickBooks is not probed and not named anywhere in the output.

## What a failure produces

Three facts, in this order. Vague is useless.

1. **Which connector.** Name it.
2. **How long it has been silent.** Age of the last good signal, in plain language: "no mail in 6 days."
3. **The fix, named.** "Re-authorize QuickBooks in connector settings." Not "an error occurred."

## How each surface shows it

**Daybook.** A red band in the right column of band 2, one row per failed connector. The band does not render when everything is healthy — an all-clear banner every morning trains the user to stop reading banners. A muted connector status row sits in the footer instead, so the healthy case is still checkable without shouting.

**Check-in, in chat.** Failures print as the first line of the today view, before meetings, in the same three-fact form. Healthy connectors get no line. This replaces the older "open with 'Gmail unavailable' and continue" rule with a probe that runs whether or not the source was reached during the check-in itself — the old rule only caught a source that threw while being read, which is exactly the failure a silently-empty connector does not produce.

## Degradation

A failed probe never blanks the tile that depends on it, and never lets it show a zero.

The tile renders its last known value with a muted "as of <when>" note. If there is no last known value, it renders "unavailable" — never "0". A zero the OS did not measure is the specific way a dashboard lies.
