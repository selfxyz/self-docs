# Core concepts

A few objects, and you have the whole model.

## Organization

Your workspace. It owns everything else: products, flows, API keys, webhooks, members, and billing. A person can belong to more than one org. Each member has a role, **owner**, **admin**, or **member**, which decides who can manage keys, webhooks, and billing. See [People](../dashboard/people.md).

## Product

The kind of verification you run: **Pre KYC**, **Age Verification**, or **Proof of Human**. The product decides what the user proves and which rules you can set. See [Configure a product](../dashboard/configure-a-product.md).

## Flow

A published, versioned configuration of one product (the dashboard calls it a *config*). It pins the product and its rules. Once published it's immutable, to change it you archive it and create a new one. You pass its `flowId` to `sessions.create(...)`. See [Anatomy of a flow](../flows/anatomy.md).

## Session

One verification attempt: one user, one flow.

```
pending → valid | invalid | error | expired
```

On any terminal status we record it in your [activity log](../dashboard/activity-log.md) and fire the `verification.completed` webhook. Each session has a credit cost, fixed when it is created.

## API key

A bearer secret the SDK uses, scoped to one org and one environment:

* `sk_test_…`: test flows, mock passports, never billed.
* `sk_live_…`: production, real proofs, real credits.

Test and live are fully isolated; the prefix decides which one a request hits. You generate keys under **Developer → API keys**. See [API keys](../dashboard/api-keys.md).

## Webhook endpoint

An HTTPS URL you register under **Developer → Webhooks** to receive signed events like `verification.completed`. We deliver with automatic retries; verify the signature with the SDK. See the [event catalog](../webhooks/events.md).

## Related

* [How verification works](how-it-works.md): the zero-knowledge model underneath.
* [Anatomy of a flow](../flows/anatomy.md): rules, documents, and versions in detail.
* [Billing](../billing/credits-and-usage.md): credits and metering.
