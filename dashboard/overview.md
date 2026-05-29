# Dashboard overview

The dashboard at [dashboard.self.xyz](https://dashboard.self.xyz) is where you configure products, manage keys, watch traffic, and pay for what you use. This page is a map.

> 📸 _**Screenshot:** Dashboard home view, with the org switcher visible in the top-left and the main nav (Home / Products / Settings)._

## Layout

```
┌───────────────────────────────────────────────────────────┐
│ [Org switcher ▾]            Home  Products  Settings      │
├───────────────────────────────────────────────────────────┤
│                                                           │
│                       (selected view)                     │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

The **org switcher** in the top-left scopes everything below it. If you belong to multiple orgs, switch here.

## Top-level sections

### Home

The landing surface. Shows recent activity across your products, quick links to common actions, and resource cards (docs, community, status).

### Products

Each product (today: Self Pass) gets its own area:

* **Landing**: list of your flows for this product, plus a button to create a new one.
* **Editor**: per-flow configuration, split into tabs:
  * [Configure](configure-a-product.md): rules, documents, settings.
  * [Deploy](publish-a-flow-version.md): preview the draft, publish a new version.
  * [Activity log](activity-log.md): verifications, sessions, errors over time.

### Settings

Org-wide configuration:

* [Account](#account): org name, deactivation.
* [People](people.md): members and pending invites.
* [API keys](api-keys.md): issue, scope, rotate, revoke.
* [Webhooks](webhooks.md): endpoints, event subscriptions, signing secrets, delivery history.
* [Billing](billing.md): plan, payment method, credit balance, invoices.

## Account

**Settings → Account** controls org-level identity:

* **Organization name**: what your teammates see in the switcher.
* **Deactivate organization**: soft-deletes the org. Flows stop accepting sessions, API keys are revoked, and webhook deliveries stop. Audit records and invoices are preserved.

> Deactivation is reversible by Self support but not self-serve. Don't deactivate to "pause". Use API key rotation or disable individual flows instead.

## Permissions

Today all members of an org have full access. Per-role permissions are on the roadmap; until then, treat membership as admin-equivalent and use API keys for service identity.
