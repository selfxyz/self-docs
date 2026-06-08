# Activity log

The **Activity log** tab on a product shows the verifications that ran against its active configuration. It's read-only, fed live from your verification records.

![Activity log](../../.gitbook/assets/activity-log.png)

It's empty until the configuration is published and traffic starts arriving.

## What you see

Finished verifications, grouped into two collapsible sections, **Passed** and **Failed**. Pending verifications are not shown; only completed ones appear. A `valid` result lands in **Passed**; `invalid`, `error`, and `expired` land in **Failed**.

Each row shows:

| Field | Meaning |
| --- | --- |
| Document | The credential type used (passport, Aadhaar, KYC). |
| Security level | The assurance level the configuration required (Standard or High Security). |
| Environment | `test` or `production`. |
| Time | When the verification finished. |
| ID | The verification's identifier. |

The pass or fail outcome isn't a column; it's the **Passed** or **Failed** section the row sits in.

## What you won't see

* Personal data the user did not disclose. If your flow asks for an age threshold, you see the pass or fail result, not the date of birth.
* The raw document or proof. Self verifies it; you never see the underlying scan.

## Getting more

* **Real-time:** subscribe to [webhooks](webhooks.md) instead of watching the log.
* **Programmatic:** read a single verification with the SDK's [`sessions.get(...)`](../sdk/nodejs.md).
* **Export:** a full, exportable record of activity and changes lives under **Settings → Audit**.
