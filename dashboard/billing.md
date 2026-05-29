# Billing (dashboard)

**Settings → Billing**. The plan card, payment method, credit balance, and invoices.

For the billing model and how to estimate cost, see [Billing](../billing/plans.md).

> 📸 _**Screenshot:** Billing page showing the plan card, credit balance, and usage chart._

## What's on this page

### Plan card

Your current plan (Free, Pro, or Enterprise; see [Plans](../billing/plans.md)) and a **Change plan** button. Changing plan opens a confirmation dialog showing what's changing (limits, included credits, monthly minimum if any) and pro-rates the switch.

### Payment method

A card on file with our payment processor. **Update** opens a secure checkout flow where you can replace it. We don't store card data ourselves; only a customer reference.

For ACH or invoiced billing (Enterprise plans), contact billing@self.xyz.

### Credit balance

Your current credit balance, what you've been granted minus what you've consumed. Verifications draw from this balance at session creation time. See [Credits and usage](../billing/credits-and-usage.md) for the rules.

If your balance approaches zero on a metered plan, you'll see a warning banner here and at the top of the dashboard. Set up usage alerts under **Notifications** to get an email or webhook before you hit zero.

### Usage charts

* **Verifications by product**: stacked area, last 30 / 90 / 365 days.
* **Credits consumed**: credits drawn down per day.
* **Cost**: current month's spend in USD (the credit-to-USD exchange rate is set on your plan's rate card).

### Invoices

Past invoices, downloadable as PDF. Each shows:

* Period covered.
* Plan fee (if any).
* Metered usage by product.
* Tax (where applicable).

## Notifications

Configure billing-related notifications:

* **Low balance**: email when credits dip below a threshold.
* **Invoice issued**: email when a new invoice posts.
* **Payment failed**: email when we can't charge the card on file.

## Related

* [Plans](../billing/plans.md).
* [Credits and usage](../billing/credits-and-usage.md).
* [Invoices](../billing/invoices.md).
