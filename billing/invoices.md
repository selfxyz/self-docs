# Invoices

We bill monthly. Invoices land in **Settings → Billing → Invoices** and (optionally) by email.

## What's on an invoice

For each billing period:

| Line | What it is |
| --- | --- |
| Plan fee | Fixed monthly fee for Pro / Enterprise. Pro-rated on mid-cycle changes. |
| Included credits | The credit grant for the period, at $0, shows what was bundled in your plan. |
| Overage usage | Per-product breakdown of metered consumption beyond the bundle. |
| Discounts | Promo credits, contract discounts, manual adjustments. |
| Tax | Where applicable (US sales tax, EU VAT, etc.). |
| **Total** | What we'll charge. |

## How charging works

* **Cards**: auto-charged on invoice date. Failed charges retry on a backoff schedule and you'll see a "Payment failed" banner in the dashboard until resolved.
* **ACH**: auto-debited where mandate allows; otherwise net-30 terms.
* **Invoiced** (Enterprise), net-30 by default, configurable.

## Failed payments

If a card charge fails:

1. We retry automatically for 7 days.
2. After day 7, your account moves to **payment hold**: live API keys stop working, webhooks stop firing, the dashboard nags. Test environment remains functional.
3. Once the invoice clears, everything resumes within minutes.

To avoid this: keep card on file current, and set up **Notifications → Payment failed** so you get an email immediately.

## Downloading invoices

Each row has a **Download PDF** action. PDFs are the legal artifact. They're what you give to your accounts payable team and your tax filings.

The CSV export (next to the PDF link) gives you the per-product, per-day breakdown for the period. Useful for reconciling against your own metering.

## Disputes

If an invoice looks wrong:

1. Open the invoice and check the line items.
2. Cross-reference with the [Activity log](../dashboard/activity-log.md) for the same period.
3. If the numbers don't match, email billing@self.xyz with the invoice number and what you expected.

We'll investigate and credit or reissue as needed. Disputes raised within 60 days of issuance are routine; older invoices are harder to amend.

## Tax

* **US**: sales tax applied based on your billing address (where applicable).
* **EU**: VAT applied unless you provide a valid VAT ID under **Settings → Billing → Tax details**.
* **Other**: depends on jurisdiction; check with your CSM if your tax setup is unusual.

A correctly-entered VAT ID can dramatically change your invoice. Set it before your first paid cycle.

## Related

* [Plans](plans.md).
* [Credits and usage](credits-and-usage.md).
* [Dashboard: Billing](../dashboard/billing.md).
