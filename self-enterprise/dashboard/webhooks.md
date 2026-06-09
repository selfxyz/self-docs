# Webhooks (dashboard)

**Developer → Webhooks**. Register the HTTPS endpoints Self posts events to. For the delivery model, signatures, retries, and payloads, see the [Webhooks section](../webhooks/overview.md).

![Webhooks settings](../../.gitbook/assets/webhook.png)

## Add an endpoint

Click **Add webhook** and fill in:

* **Name**: a label for the endpoint.
* **Webhook URL**: the **full** public HTTPS URL including your handler's path, e.g. `https://yourapp.com/webhooks/self`, not just the domain.
* **Environment**: `test` or `live` (defaults to test).

{% hint style="warning" %}
Always include the path. If you register a bare domain (`https://yourapp.com`), deliveries often land on your web app's page route instead of your webhook handler, where they're silently swallowed.
{% endhint %}

Click **Save**. The endpoint is saved right away and the **signing secret** (`whsec_...`) is revealed **once** in a banner. This is the only time you'll see it, copy it into your secret manager as `SELF_WEBHOOK_SECRET`.

Every endpoint receives the `verification.completed` event; there's no per-endpoint event picker. Branch on `event.type` in your handler. See the [event catalog](../webhooks/events.md).

## Verify the signature

Once you have the `whsec_...` secret, wire up real signature verification so you only act on genuine events. See [Verify webhooks](../sdk/verify-webhooks.md).

## Send a test request

A saved endpoint has a **Send Test Request** button. It delivers a sample event to your URL and reports whether the endpoint accepted it (returned `2xx`). It's repeatable, use it to confirm your handler is reachable and responding before you rely on live traffic. Sending a test is optional and never required to save an endpoint.

## Managing endpoints

Each saved endpoint shows its name, URL, and environment. You can:

* **Delete** it, which stops delivery to that URL.
* **Rotate** its signing secret. The new `whsec_...` is revealed once, the same way it was at creation. Roll it into `SELF_WEBHOOK_SECRET` and redeploy.

## Related

* [Webhooks: overview](../webhooks/overview.md): delivery, retries, and idempotency.
* [Signature verification](../webhooks/signature-verification.md): how to validate a delivery.
* [Event catalog](../webhooks/events.md): every event and its payload.
