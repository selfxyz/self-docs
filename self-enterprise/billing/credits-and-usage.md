# Credits and usage

## Credits

Usage is measured in **credits**, an abstract unit. How many you have depends on your plan:

* **Free**: a one-time grant that never renews.
* **Starter**: a monthly allotment that resets each billing cycle.
* **Enterprise**: custom grants.

See [Plans](plans.md).

## Per-verification cost

Each verification costs a fixed number of credits, decided by the product and set when the session is created (not at completion):

| Product | Credit cost |
| --- | --- |
| Pre KYC | 25 |
| Age Verification | 10 |
| Proof of Human | 10 |

The dashboard shows the cost when you configure a product. A session is charged its cost at creation, and an in-flight session keeps the price it was created with. To estimate spend, multiply your expected verifications by the product's cost.

## What you're charged for

You're billed for verifications that finish, not for sessions the user abandons:

| Session result | Charged? |
| --- | --- |
| `valid` | Yes |
| `invalid` (a rule failed) | Yes |
| `error` (technical failure) | No |
| `expired` (never completed) | No |

## Insufficient credits

By default Self uses a **hard credit gate**: if your balance can't cover a session's cost, `sessions.create(...)` is rejected with HTTP `402` (the SDK throws `SelfApiError` with `statusCode: 402`, and `details` carries `balance`, `required`, and `planTier`). The session is not created.

To add capacity, upgrade your plan from **Manage plan** (or talk to sales about Enterprise). Watch the credits meter on the **Settings → Usage & Billing** tab so this doesn't surprise you.

## Test environment

Test verifications (sessions created with an `sk_test_` key) never consume credits. Use test freely.

## Related

* [Plans](plans.md): what each tier includes.
* [Dashboard: Billing](../dashboard/billing.md): the Usage & Billing tab.
