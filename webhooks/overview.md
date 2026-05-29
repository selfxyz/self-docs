# Webhooks: overview

Webhooks are the canonical way to learn that a verification finished. Polling works but it's lossy under retries and expensive at scale.

## How they work

1. You register an HTTPS endpoint in the dashboard ([Settings → Webhooks](../dashboard/webhooks.md)).
2. You subscribe to one or more event types.
3. When the event fires, we POST a JSON payload to your endpoint, signed so you can verify it came from us.
4. Your handler verifies the signature (via the SDK) and processes the event.
5. You return `2xx` to acknowledge. Non-2xx triggers a retry.

```
┌─────────────────┐  fire   ┌─────────────────┐  POST   ┌──────────────┐
│ verifier finish │ ──────▶ │ webhook         │ ──────▶ │ your handler │
│                 │         │ dispatcher      │         │              │
└─────────────────┘         └─────────────────┘  2xx   └──────────────┘
                                    ▲                          │
                                    └─── retry on non-2xx ─────┘
```

## What we send

A single POST with a JSON body. The body is a discriminated-union on `type`:

| Event type | Fires when |
| --- | --- |
| `verification.completed` | Off-chain verification finishes (success or failure). |
| `verification.storage_committed` | Async storage write succeeded. |
| `verification.storage_failed` | Async storage write permanently failed. |

See [Event catalog](events.md) for the full payloads.

## Headers

Every delivery carries these signature headers:

```
svix-id: msg_2abc...               # unique message ID
svix-timestamp: 1716998400         # unix seconds
svix-signature: v1,base64...       # HMAC signature
content-type: application/json
```

Optionally, on replays:

```
svix-replay: true
```

## Ordering and delivery guarantees

* **At-least-once.** A successful proof can produce multiple deliveries, retry duplicates, network blips, manual replays. Make your handler idempotent.
* **No strict ordering.** `verification.completed` and `verification.storage_committed` are independent. Don't assume one arrives before the other.
* **Bounded latency.** Typically sub-second from verification to delivery. Under retry backoff, latency can grow to minutes.

## Idempotency

The `svix-id` header is unique per delivery. The event payload itself uses a stable ID:

* Terminal events (`verification.completed`): `<verification_id>-<status>`.
* Storage events: `<verification_id>-storage-<state>`.

Dedupe in your handler using the event ID. See [Best practices](best-practices.md).

## Reliability

* We retry failed deliveries on an exponential schedule for up to several days.
* You can replay any past event from the dashboard.
* Failed deliveries (after all retries) park in **Settings → Webhooks → Failed deliveries**.

## Related

* [Signature verification](signature-verification.md): how to validate a delivery.
* [Event catalog](events.md): every event we send, with payloads.
* [Best practices](best-practices.md): idempotency, ordering, replay.
* [SDK: Verify webhooks](../sdk/verify-webhooks.md): the easy path in Node.
