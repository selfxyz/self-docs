# Best practices

A small list of things that prevent the common production fires.

## 1. Make handlers idempotent

We deliver at-least-once. Network blips, our own retries, and manual replays all produce duplicates. Your handler should treat duplicates as a non-event.

### Dedup by event ID

The simplest pattern:

```ts
const eventId = `${event.verification_id}-${event.type}`;

const inserted = await db.query(
  `INSERT INTO processed_events (event_id) VALUES ($1) ON CONFLICT DO NOTHING RETURNING 1`,
  [eventId],
);

if (inserted.rowCount === 0) {
  return;   // already processed
}

await applyVerification(event);
```

Use the `verification_id` (stable across retries), not the `svix-id` (unique per delivery, including replays).

## 2. Acknowledge quickly, work async

Return `2xx` within a few seconds. If your handler does expensive work, database fan-out, downstream API calls, emails, push to a queue and return immediately.

```ts
app.post('/webhooks/self', express.raw({ type: 'application/json' }), async (req, res) => {
  const event = SelfWebhooks.verify(req.body, req.headers, secret);
  await jobQueue.enqueue('process-verification', event);
  res.status(200).end();
});
```

Slow handlers get retried, which produces duplicates, which compounds. Fast ack + queue is the only stable pattern.

## 3. Don't trust ordering

`verification.completed` and `verification.storage_committed` are independent. You may see `storage_committed` before you've finished processing `completed` (or vice versa).

Handle each event in isolation. If you need to know "did storage commit for verification X", query the API, don't infer from the event sequence.

## 4. Distinguish status carefully

`verification.completed` fires with four statuses. Don't lump them:

| Status | What it means | Common handling |
| --- | --- | --- |
| `valid` | Proof verified, predicates passed. | Mark user verified, unlock the gated feature. |
| `invalid` | Proof verified, a predicate failed. | Tell the user *why* (e.g. age requirement); offer retry. |
| `error` | Technical failure (bad proof, unsupported document). | Tell the user to retry; alert ops if rate spikes. |
| `expired` | Session timed out. | Generate a new session if the user is still around. |

A common bug: treating any non-`valid` as a final rejection. `invalid` and `error` are recoverable; `expired` definitely is.

## 5. Verify before doing anything

Don't read `event.type` until `SelfWebhooks.verify(...)` has returned. An attacker can send any body to your endpoint; only the signature proves it came from us.

```ts
// Wrong:
app.post('/webhooks/self', (req, res) => {
  const event = JSON.parse(req.body);     // ← trusting unverified body
  if (event.type === 'verification.completed') {
    grantAccess(event.external_uuid);     // ← attacker can call this
  }
});

// Right:
app.post('/webhooks/self', express.raw({ type: 'application/json' }), (req, res) => {
  const event = SelfWebhooks.verify(req.body, req.headers, secret);
  // ...now safe to act on event
});
```

## 6. Return the right status codes

| Your response | What we do |
| --- | --- |
| `2xx` | Delivery successful. No retry. |
| `4xx` (except 408, 429) | Treated as a permanent rejection. No retry. |
| `408`, `429`, `5xx` | Retried with backoff. |
| Timeout / connection refused | Retried with backoff. |

So:

* Return `400` on signature failure. You want it dropped, not retried.
* Return `500` (or just throw) on transient errors. You want it retried.
* Don't return `400` because of a database hiccup. You'll silently drop events.

## 7. Use one endpoint per environment

Don't share a single endpoint between staging and production. They'll have different secrets and different traffic profiles. Register `https://api.example.com/webhooks/self` for prod and `https://api-staging.example.com/webhooks/self` for staging, each with its own subscription.

## 8. Watch the failure queue

Set up an alert when **Settings → Webhooks → Failed deliveries** grows. A handler that's been silently failing for two days is a known production incident pattern.

## 9. Replay safely

When you ship a fix and want to drain failed deliveries, you can:

1. **Replay all** from the dashboard. Fastest, but be sure your handler is ready.
2. **Replay one, observe, then replay the rest**. Safer if you've been broken for a while.

Replays carry `svix-replay: true` if you want to log them distinctly. Your handler should be idempotent enough that you don't need to handle replays specially.

## 10. Log the request ID

Every delivery includes `svix-id`. Log it on every handler invocation:

```ts
log.info({ msg: 'webhook_received', svixId: req.headers['svix-id'], type: event.type });
```

When you ask support about a missing or wrong delivery, that's the ID to quote.
