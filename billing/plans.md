# Plans

Three tiers. Pick based on volume and support needs. Plans are configured under [Settings → Billing](../dashboard/billing.md) in the dashboard.

> Concrete prices and limits live in the dashboard and on [self.xyz/pricing](https://self.xyz/pricing). The shape below describes what each plan unlocks, not the exact dollar amounts.

## Free

For evaluation, prototypes, and side projects.

* **Included credits**: a monthly allotment, enough to integrate end-to-end and run small-volume traffic.
* **Rate limits**: modest; sufficient for a single backend.
* **Support**: community (Discord, GitHub issues).
* **Features**: full Self Pass capability, test + live environments, webhooks, audit log.

Free is meant to be a real on-ramp, not a marketing tease. If you outgrow it, you'll see the next-tier prompts in-product.

## Pro

For production workloads with modest volume.

* **Monthly fee**: fixed.
* **Included credits**: much larger pool than Free; spillover billed per-credit.
* **Rate limits**: production-grade.
* **Support**: email, business-hours SLA.
* **Features**: everything in Free, plus usage alerts, exportable audit log, multiple webhook endpoints per env.

## Enterprise

For volume, regulated environments, and bespoke needs.

* **Pricing**: negotiated; usage-based with monthly minimums.
* **Rate limits**: negotiated.
* **Support**: dedicated channel (Slack / Teams), 24/7 incident response, named CSM.
* **Features**: everything in Pro, plus:
  * **SSO / SCIM**: enforced sign-in and directory provisioning via your identity provider (SAML, OIDC, Google Workspace, Okta).
  * **Custom data residency**: EU / US / region of your choice for verification storage.
  * **Custom contracts**: DPA, BAA, security questionnaires.
  * **Direct line to engineering**: feature requests routed through your CSM.

Contact sales@self.xyz to start an Enterprise conversation.

## Switching plans

In **Settings → Billing → Change plan**:

* **Upgrading**: change takes effect immediately. You're pro-rated for the remainder of the cycle.
* **Downgrading**: change takes effect at the end of the current cycle. Existing flows and webhooks continue working; you can't downgrade below your committed minimum on Enterprise.
* **Pausing**: not directly supported. The closest is downgrading to Free; usage continues at Free limits.

## What you don't pay for

* Test-environment verifications. Test traffic doesn't bill.
* Webhook deliveries. Bundled into the per-verification cost.
* Audit log storage and dashboard usage.
* Sessions that never complete (e.g. `expired`, or the user abandons after creation). You only pay for verifications that finish with a non-`expired` status. See [Credits and usage](credits-and-usage.md).

## Related

* [Credits and usage](credits-and-usage.md): how the per-verification cost is computed.
* [Invoices](invoices.md).
* [Dashboard: Billing](../dashboard/billing.md).
