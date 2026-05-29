# Activity log

The **Activity log** tab in the product editor shows every session against a flow, plus aggregate signal at the top.

> 📸 _**Screenshot:** Activity log with the sessions-over-time chart at top and the paginated session table below._

## What you see

Top of the tab:

* **Sessions over time**: line chart, last 7 / 30 / 90 days.
* **Success rate**: `completed / (completed + failed + expired)`.
* **Average time to completion**: from session creation to webhook fire.

Below that, a paginated table:

| Column | Meaning |
| --- | --- |
| Session ID | Internal verification ID. Click to drill in. |
| External UUID | Your identifier (e.g. `user_42`) passed to `sessions.create(...)`. |
| Status | `pending`, `completed`, `failed`, `expired`. |
| Document | The credential type used (passport / Aadhaar / KYC). |
| Country | Issuer country of the credential. |
| Created at | When the session was opened. |
| Completed at | When the proof landed (blank if not yet completed). |
| Version | Flow version pinned to this session. |

## Drilling in

Clicking a row opens the session detail:

> 📸 _**Screenshot:** A session detail panel showing `proofAttributes`, the rules version, and webhook delivery state._


* The disclosed `proofAttributes` (only the attributes the user proved).
* The exact rules version evaluated.
* Webhook delivery state, which subscriptions received which events, and the response codes.
* If `failed`, the failure reason (`predicate_not_met`, `unsupported_document`, `expired`, `signature_invalid`, etc.).

## Exporting

Use the **Export** button to download a CSV of the current filter window. Webhook delivery details are not included in the export; use the [Webhooks](webhooks.md) tab for that.

## What you won't see

* Personal data the user did not disclose. If your flow asks for `age >= 18`, you see the boolean result, not the date of birth.
* Credentials themselves. Self verifies the proof; you never see the underlying passport scan.

## Related

* [SDK: sessions.get(...)](../sdk/nodejs.md#get): same data, programmatic access.
* [Webhooks](webhooks.md): for real-time delivery instead of polling the log.
