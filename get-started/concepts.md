# Concepts

The model is small. Five things to know.

## Organization

The top-level tenant. Flows, API keys, webhook subscriptions, members, and billing all hang off an org. A user can belong to multiple orgs; you switch between them in the dashboard's top-left selector.

Created on sign-up. Renamed and deactivated under **Settings → Account**.

## Product

A product is a verification capability, currently **Self Pass** (passport / national ID / KYC). Each product has its own configuration model and its own per-verification cost.

You configure products at the org level. A single org can run multiple products in parallel.

## Flow

A flow is a published, versioned configuration for a product. It pins:

* **Rules**: the predicates the user's proof must satisfy (`age >= 18`, `nationality in [...]`, etc.).
* **Documents**: which credential types are acceptable.
* **Settings**: success URL, failure URL, branding.

Flows are versioned. Editing creates a draft; publishing freezes the draft as a new version and points the flow's `latestPublishedVersionId` at it. Older versions stay queryable for audit.

A `flowId` is what you pass to `sessions.create(...)`.

## Session (a.k.a. verification)

A session is a single attempt at verifying one user against one flow.

It has a lifecycle:

```
created → pending (user opened URL) → completed | failed | expired
```

When `completed`, a `verification` record is finalized in your audit log with the proof attributes the user disclosed. We fire `verification.completed` on your webhook.

Sessions also have a per-session **cost** in credits, baked in at creation time. See [Credits and usage](../billing/credits-and-usage.md).

## API key

A Bearer credential used by the Enterprise SDK to authenticate. Issued in two flavors:

* `sk_test_...`: test environment. Talks to test flows only. Mock passports accepted. No credits charged.
* `sk_live_...`: production. Real proofs. Real credits.

Keys are scoped to an org. Issue, rotate, and revoke under **Settings → API keys**.

## Webhook subscription

A signed delivery channel for events. You register a URL in **Settings → Webhooks** and pick which event types you want. We send signed JSON payloads with automatic retries.

See the [event catalog](../webhooks/events.md) for what's available.

## Related concepts

* [Anatomy of a flow](../flows/anatomy.md): deep dive into rules, documents, and settings.
* [Dashboard: API keys](../dashboard/api-keys.md): how Bearer keys map to environments.
* [Billing](../billing/credits-and-usage.md): credits, plans, metering.
