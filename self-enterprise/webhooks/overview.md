# Webhooks: overview

Webhooks are the canonical way to learn that a verification finished. Polling works but it's lossy under retries and expensive at scale.

## How they work

1. You register an HTTPS endpoint in the dashboard ([Developer → Webhooks](../dashboard/webhooks.md)) and get the signing secret when you save it. (You can send a test request afterwards to confirm it's reachable.)
2. When an event fires, we POST a JSON payload to your endpoint, signed so you can verify it came from us.
3. Your handler verifies the signature (via the SDK) and processes the event.
4. You return `2xx` to acknowledge. A non-2xx (or timeout) triggers a retry.

```
┌──────────────┐   1. signed POST (event)     ┌──────────────────┐
│     Self     │ ───────────────────────────▶ │  Your endpoint   │
│              │                              │  verify the sig, │
│              │ ◀─────────────────────────── │  then return 2xx │
└──────────────┘   2. ack (2xx)               └──────────────────┘

No 2xx, or a timeout? Self retries the delivery with backoff.
```

## What we send

A single POST with a JSON body, typed as a discriminated union on `type`. One event is delivered today:

| Event type | Fires when |
| --- | --- |
| `verification.completed` | Off-chain verification finishes (success or failure). |

See [Event catalog](events.md) for the full payload.

## Ordering and delivery guarantees

* **At-least-once.** The same event can arrive more than once (a retry after a failed delivery, a network blip). Make your handler idempotent.
* **Bounded latency.** Typically under a second from verification to delivery. Under retry backoff, latency can grow to minutes.

## Idempotency

The event carries a stable ID derived from the verification, so retries of the same event share it:

* `verification.completed`: `<verification_id>-completed`

Dedupe in your handler on `verification_id`. See [Best practices](best-practices.md).

## Reliability

Failed deliveries (non-2xx, timeout, or connection error) are retried automatically on an exponential schedule over a span of days, so a brief outage on your side recovers on its own.

## Related

* [Signature verification](signature-verification.md): how to validate a delivery.
* [Event catalog](events.md): the event we send, with its payload.
* [Best practices](best-practices.md): idempotency and status handling.
* [SDK: Verify webhooks](../sdk/verify-webhooks.md): the easy path in Node.
