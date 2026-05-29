# Credits and usage

How we meter what you use and turn it into a bill.

## The credit model

Usage is denominated in **credits**, an abstract unit. Each plan includes some credits per month, and your USD price-per-credit is set on your plan's rate card.

> Credits are not USD cents. The exchange rate is part of your contract; check **Settings → Billing → Plan card** for your current rate.

This lets us:

* Adjust per-product pricing without touching your contract.
* Run promos and waivers in credits.
* Set spending caps independently of session volume.

## Per-product cost

Each product has a per-verification credit cost, baked in at session creation time (not at completion):

| Product | Credit cost (illustrative) |
| --- | --- |
| Self Pass, biometric passport | 10 |
| Self Pass, Aadhaar | 5 |
| Self Pass, KYC attestation | 12 |

(Concrete costs live in your dashboard. The numbers above are illustrative.)

The cost is debited from your credit balance the moment a session is created, with a hold. If the session expires unused, the hold is **released**, you only pay for verifications the user actually completed.

## What counts as "consumed"

| Session terminal status | Consumed? |
| --- | --- |
| `valid` | Yes |
| `invalid` (predicate failed) | Yes |
| `error` (technical failure) | No, automatically refunded |
| `expired` (user never finished) | No, automatically refunded |

You pay for verification work, not for sessions the user never got to.

## Credit balance lifecycle

```
[plan grant @ cycle start] → [debited on session create] → [refund on expired/error]
                                         │
                                         ▼
                          [topped up via overage purchase, if enabled]
```

Watch the balance in **Settings → Billing → Credit balance**. It auto-refills at the start of each cycle.

## Insufficient credits

If your balance is too low to cover a session's cost, `sessions.create(...)` throws `SelfApiError` with `code: 'insufficient_credits'`. The session is not created and nothing is held.

To avoid this in production:

* Set up **low-balance alerts** under **Notifications**.
* Configure **overage** so we automatically purchase top-up credits when you cross zero (available on Pro and Enterprise plans).
* Watch the **Credits consumed** chart in Billing to forecast burn.

## Reading usage programmatically

Metered events are aggregated by our usage system. Customers on Enterprise plans get programmatic access to a usage API; reach out to your CSM.

For Pro and Free, the dashboard's CSV export is the source of truth.

## Test environment

Test verifications never consume credits and never appear on invoices. They're not even tracked against rate-limit windows. Use test freely.

## Examples

### Sizing for a launch

You expect 50,000 verifications in the first month, mostly biometric passport (cost 10 credits each):

* Expected consumption: 50,000 × 10 = 500,000 credits.
* If your plan includes 200,000, you'll need 300,000 in overage, verify your overage rate before launch.

### Diagnosing a spike

The **Credits consumed** chart shows daily burn. If you see a spike, drill into the [Activity log](../dashboard/activity-log.md) for the same day, the External UUID column usually identifies the culprit (a stuck retry loop, a bot, a buggy integration).

## Related

* [Plans](plans.md).
* [Invoices](invoices.md).
* [Dashboard: Billing](../dashboard/billing.md).
