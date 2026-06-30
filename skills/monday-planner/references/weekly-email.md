# Weekly email — generic default and per-engagement templates

The planner's Output 3 drafts a weekly check-in email. It works out of the box with a generic template, and any engagement can override it with its own copied, customized template.

## Two modes

**Generic (default).** If no engagement is configured for a weekly email, the planner produces one neutral weekly summary in the user's voice from `templates/weekly_email_TEMPLATE.html`, leaves the recipient blank, and saves it to the user's project folder as `Weekly_Email_<M_D>_v1.html`. Nothing to configure.

**Per-engagement.** An engagement gets its own email when both are true:

1. Its row in the Engagements database has **`Weekly report` = checked** (the checkbox that exists for exactly this purpose).
2. Its entry in `engagements[]` in `solo-os-config.json` has a **`weekly_email`** block.

When flagged, the planner drafts one email per such engagement, filling that engagement's own template and using its recipients and save path. This is how a client-specific format (custom branding, named distribution list, a recurring "week N" block) stays out of the core skill and lives with the engagement instead.

## Config shape

Add a `weekly_email` block to the engagement in `solo-os-config.json`:

```json
{
  "name": "Example Client ERP Migration",
  "qb_customer": "Example Client",
  "rate": 125,
  "weekly_email": {
    "template_path": "Projects/Example Client/templates/weekly_email.html",
    "to": "erp-team@exampleclient.com",
    "cc": "you@yourfirm.com",
    "subject_format": "{date} Weekly Check-in - {theme}",
    "save_to": "Projects/Example Client/deliverables/WeeklyCheckin_{date}.html"
  }
}
```

Field notes:

- `template_path` — path to this engagement's copy of the template. Start from `templates/weekly_email_TEMPLATE.html`, copy it into the engagement's own folder, then customize freely.
- `to` / `cc` — recipient lines. Omit `to` to leave it blank for manual fill.
- `subject_format` — supports `{date}` (the Monday date) and `{theme}` (the week theme).
- `save_to` — where the draft HTML is written; supports `{date}`. Defaults to the user's project folder if omitted.

## Setting up an engagement template

1. Copy `templates/weekly_email_TEMPLATE.html` into the engagement's project folder.
2. Customize the styling and sections — branding, colors, any client-specific blocks. Keep the `{{TOKEN}}` placeholders you still want the planner to fill (see the comment at the top of the template for the token list).
3. Add the `weekly_email` block above to that engagement in config, pointing `template_path` at your copy.
4. Check `Weekly report` on the engagement's row in the Engagements database.

The planner will draft that engagement's email every Monday, in the user's voice, for review and manual send. It never sends.

## Note on the original OFP format

The OFP "Monday Check-in" (teal week-N block, OFP ERP team distribution, Calibri/Aptos styling) is simply the first engagement template — it lives in the OFP project folder and is wired in through that engagement's `weekly_email` block. It is engagement data, not part of the shipped product, so it is not committed to this repo.
