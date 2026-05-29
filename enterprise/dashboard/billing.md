# Billing (dashboard)

**Settings → Billing**. The plan card, payment method, credit balance, and invoices.

For the billing model and how to estimate cost, see [Billing](../billing/plans.md).

## What's on this page

### Plan card

Your current plan (Free, Pro, Enterprise — see [Plans](../billing/plans.md)) and a **Change plan** button. Changing plan opens a confirmation dialog showing what's changing (limits, included credits, monthly minimum if any) and pro-rates the switch.

### Payment method

A Stripe-managed card. **Update** opens a Stripe Checkout session where you can replace it. We don't store card data ourselves; Stripe holds the card and we hold a customer reference.

For ACH or invoiced billing (Enterprise plans), contact billing@self.xyz.

### Credit balance

Your current credit balance — what you've been granted minus what you've consumed. Verifications draw from this balance at session creation time. See [Credits and usage](../billing/credits-and-usage.md) for the rules.

If your balance approaches zero on a metered plan, you'll see a warning banner here and at the top of the dashboard. Set up usage alerts under **Notifications** to get an email or webhook before you hit zero.

### Usage charts

* **Verifications by product** — stacked area, last 30 / 90 / 365 days.
* **Credits consumed** — credits drawn down per day.
* **Cost** — current month's spend in USD (the credit-to-USD exchange rate is set on your plan's rate card).

### Invoices

Past invoices, downloadable as PDF. Each shows:

* Period covered.
* Plan fee (if any).
* Metered usage by product.
* Tax (where applicable).

## Notifications

Configure billing-related notifications:

* **Low balance** — email when credits dip below a threshold.
* **Invoice issued** — email when a new invoice posts.
* **Payment failed** — email when Stripe can't charge the card on file.

## Related

* [Plans](../billing/plans.md).
* [Credits and usage](../billing/credits-and-usage.md).
* [Invoices](../billing/invoices.md).
