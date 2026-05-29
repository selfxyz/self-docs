# Webhooks (dashboard)

**Settings → Webhooks**. Manage where Self sends event notifications and watch their delivery.

For the conceptual model and signature verification, see the [Webhooks section](../webhooks/overview.md).

> 📸 _**Screenshot:** Webhooks page with a couple of registered endpoints, status badges, and recent-delivery info._

## Adding an endpoint

Click **Add endpoint**:

* **URL**: your HTTPS endpoint that will receive POSTs. Must be reachable from the public internet.
* **Description**: optional; useful when you have multiple environments registered.
* **Event types**: which events you want delivered. Subscribing to all is fine; filtering happens at delivery time.

On save, the dashboard shows the **signing secret** (`whsec_...`) **once**. Store it in your secret manager as `SELF_WEBHOOK_SECRET`.

## Endpoint list

For each endpoint:

* **Status**: `active`, `disabled`, or `failing` (the last several deliveries failed).
* **Last delivery**: timestamp + response code.
* **Subscribed events**: the filter list.

Click an endpoint to see its delivery history (response codes, latency, retry counts).

> 📸 _**Screenshot:** Endpoint detail showing the delivery history table with timestamps, response codes, and retry counts._

## Replay & test

From an endpoint's detail view:

* **Send test event**: sends a synthetic event of your choice. Useful when wiring up a new handler.
* **Replay**: re-sends a specific past event. Useful when you've fixed a bug in your handler and want to drain failed deliveries.

Replays carry a header `svix-replay: true` so you can tell them apart from originals if you want.

## Retry policy

Failed deliveries (non-2xx, timeout, connection error) are retried automatically on an exponential schedule. After the final attempt the event is parked in **Failed deliveries**; you can replay from there.

> See [Best practices](../webhooks/best-practices.md) for handler design, idempotency, ordering, and replay.

## Rotation

You can rotate the signing secret per endpoint:

1. **Rotate secret**, generates a new `whsec_...`. The old secret is shown alongside it.
2. We sign each delivery with the new secret. Verifications using either work for a 24-hour overlap window.
3. After 24 hours the old secret stops working.

Roll the new secret into your handler within that window.

## Disabling vs. deleting

* **Disable**: keeps the endpoint configured but stops delivery. Re-enable any time. Replays still possible.
* **Delete**: removes the endpoint. Pending retries are dropped. Replays not possible.

## Related

* [Webhooks: overview](../webhooks/overview.md).
* [Signature verification](../webhooks/signature-verification.md).
* [Event catalog](../webhooks/events.md).
