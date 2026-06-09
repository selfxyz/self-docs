# Dashboard overview

The [dashboard](https://dashboard.self.xyz) is where you configure products, manage keys, watch traffic, and pay for what you use. This page is a map.

![Dashboard home](../../.gitbook/assets/dashboard-home.png)

## Top-level sections

### Home

The landing surface. Shows recent activity across your products, quick links to common actions, and resource cards (docs, community, status).

### Products

Each product (Pre KYC, Age Verification, Proof of Human) has its own page and holds **one active configuration at a time**. The page splits into tabs:

* [Configure](configure-a-product.md): disclosure rules and (for Pre KYC) additional data.
* **Test** and **Live**: the per-environment integration tabs. Each shows the configuration's `flowId`, SDK snippets, and a link to your [API keys](api-keys.md). They unlock once the configuration is published.
* [Activity log](activity-log.md): verifications and errors over time.
* **settings**: archive the configuration, plus a usage and credit snapshot.

The Test, Live, and Activity log tabs stay locked until the configuration is published. To change a live configuration you archive it first, then create a new one.

### Archive

Where a product's previous configurations go once you replace them. Archived configurations have their proof requests **deactivated**, but their activity log stays viewable for audit.

### Developer

Org-scoped developer surfaces, in the sidebar under **Developer**:

* [**Webhooks**](webhooks.md): register HTTPS endpoints and manage signing secrets.
* [**API keys**](api-keys.md): generate and revoke the `sk_test_…` / `sk_live_…` keys your backend uses.
* **Developer Docs**: a link to these docs.

### Settings

Org-wide configuration, organized into tabs:

* [**General**](#general): organization name, your profile, theme, and deactivation.
* [**Usage & Billing**](billing.md): plan, credit balance, and usage.
* **Audit**: an exportable record of changes.
* [**People**](people.md): members and invites.
* **System Status**: live service status.

## Permissions

Members have one of three roles, `owner`, `admin`, or `member`, which control who can manage keys, webhooks, and billing. See [People → Roles](people.md#roles). For machine access, use [API keys](api-keys.md) rather than a member's login.
