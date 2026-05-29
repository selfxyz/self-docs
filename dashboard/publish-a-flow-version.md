# Publish a flow version

Flows are versioned. The **Deploy** tab is where you turn a draft into a published version that the API will use.

## How versioning works

* The **draft** is what you see in the Configure tab. It's mutable.
* A **published version** is immutable. Its rules, documents, and settings are frozen.
* The flow's `latestPublishedVersionId` points at the most recently published version. New sessions use that version.
* In-flight sessions stay pinned to whatever version they were created against. Publishing never disturbs them.

## Publishing

In the **Deploy** tab:

1. Review the diff: what's changed since the last published version.
2. Click **Publish version**.
3. Confirm. The draft is frozen, a new version is created, and `latestPublishedVersionId` advances.

> 📸 _**Screenshot:** Deploy tab showing the version diff and the **Publish version** button._

The next `sessions.create(...)` call against this `flowId` will use the new version.

## Rollback

There's no destructive rollback. To revert, edit the draft back to the previous state and publish again. This creates a new version whose contents match the old one. Audit history is preserved.

## Test before publishing

The Deploy tab includes a **Preview** link. It opens the hosted page against your draft so you can run through with the Self app on a mock passport. Previews do not bill credits and do not produce a `verification.completed` event on your webhook subscription.

## When to cut a new version

* Adding or removing a rule.
* Changing the document allow-list.
* Adjusting copy or branding on the hosted page.

You don't need to publish for trivial dashboard-only changes (e.g. renaming the flow in the listing).

## Related

* [Configure a product](configure-a-product.md).
* [Activity log](activity-log.md): see which version each session ran against.
