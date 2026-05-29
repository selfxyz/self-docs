---
icon: life-ring
---

# Troubleshooting

Things that commonly go wrong, and how to unstick them.

If none of these apply, email support@self.xyz with the request ID (`X-Request-Id` header on API responses, `svix-id` on webhook deliveries).

## Authentication

### `401 unauthorized`

The API key is missing, malformed, or revoked.

* Check the `Authorization: Bearer ...` header is present.
* Confirm the key starts with `sk_test_` or `sk_live_`.
* In the dashboard, check the key's **Last used at**. If it's never been used, you may be reading from the wrong env var. If it's been used recently, it may have been revoked.
* The full key is shown once on creation. If you've lost it, create a new key and revoke the old one.

### `403 forbidden`

The key is valid but isn't allowed to act on this resource. Two common cases:

* **Test key + live flow** (or vice versa). Check the flow ID is in the right environment.
* **Cross-org access.** The key belongs to org A, the session/flow belongs to org B. Confirm you're hitting the right org in the dashboard.

## API requests

### `400 validation_failed`

The request body didn't match the schema. The `details.issues` array tells you what.

```json
{
  "error": {
    "code": "validation_failed",
    "details": {
      "issues": [
        { "path": ["externalUuid"], "message": "String must contain at least 1 character(s)" }
      ]
    }
  }
}
```

Fix the field at `path`. Don't retry on `400`, the request will keep failing.

### `404 flow_not_found`

The `flowId` is wrong, points at a draft (not yet published), or the flow is archived.

* Open the flow in the dashboard. If you don't see a "Published" badge, click **Publish version**.
* If the flow was deleted, recreate it; the ID is gone.

### `402 insufficient_credits`

Your org's credit balance is too low to cover this session's cost.

* Check **Settings → Billing → Credit balance**.
* If you're on Free, the monthly grant resets at the start of the cycle. If you're on Pro/Enterprise, configure overage.
* Set a **low-balance notification** so this doesn't surprise you again.

### `429 rate_limited`

Per-API-key rate limit exceeded. Honor the `Retry-After` header.

* If your traffic legitimately exceeds the limit, contact sales@self.xyz about higher limits or move to Enterprise.
* If a single key is doing all the work, split traffic across multiple keys, limits are per-key.
* The SDK retries 429s automatically; if you're getting them, you've exhausted SDK retries too.

### `5xx` errors

Transient. Retry with exponential backoff. If it's persistent (more than 30 seconds), check [status.self.xyz](https://status.self.xyz) and contact support with the request ID.

## Webhooks

### Webhook handler never receives events

* In the dashboard, check **Settings → Webhooks → [endpoint] → Delivery history**. Are deliveries being attempted?
  * If **no attempts**: your endpoint isn't subscribed to the event type you're expecting. Check the subscription list.
  * If **attempts but failing**: see below.
* Confirm your endpoint is reachable from the public internet (not behind a VPN, no firewall blocking POST).
* If using a tunnel (ngrok, Cloudflare Tunnel), confirm the tunnel is still up.

### Webhook deliveries fail with `400`

Your handler is rejecting the delivery. Most likely:

* **Signature verification failed.** Check that:
  * You're using the *raw* body, not a JSON-parsed object.
  * The signing secret matches the dashboard's `whsec_...` for this endpoint.
  * You're not rewriting headers between proxy and handler.
* **Body schema mismatch.** Update to the latest SDK version, we may have added an event type your version doesn't know.

### Webhook deliveries fail with `5xx`

Your handler is erroring before completing. Check your logs for the handler invocation matching the `svix-id` in the failed delivery row.

### Duplicate events

Expected. We deliver at-least-once. See [Best practices](../webhooks/best-practices.md#1-make-handlers-idempotent) for dedup patterns.

## SDK

### `Cannot find module '@selfxyz/enterprise-sdk'`

The package is ESM-only and requires Node 18+.

* Confirm Node version (`node -v`).
* Confirm your `package.json` has `"type": "module"`, or you're using `.mjs` files, or your build system supports ESM.
* TypeScript: `"module": "NodeNext"` or `"ESNext"` in `tsconfig.json`.

### Webhook verification throws `WebhookVerificationError`

The body or headers were modified between Self and your handler.

* You're using `express.json()` before the webhook route, the body is now a parsed object, not the raw bytes we signed. Use `express.raw({ type: 'application/json' })` for the webhook path only.
* A proxy is normalizing or rewriting the body. Configure it to pass through untouched.
* Wrong signing secret. Each webhook endpoint has its own.

### Types don't narrow on `event.type`

```ts
if (event.type === 'verification.completed') {
  event.status;  // TS error?
}
```

* Confirm you're importing `WebhookEvent` from `@selfxyz/enterprise-sdk` (which is the discriminated union), not constructing it yourself.
* If you're using older TypeScript (<4.4), narrowing on discriminated unions may need explicit type guards.

## Mock passports

For testing in `sk_test_` environments, the Self app's "Test mode" issues mock passports you can use to drive every code path.

To enable in the Self app:

1. Open the Self app on your test device.
2. Settings → Developer mode → enable.
3. A new "Test mode" toggle appears on the home screen.
4. While Test mode is on, the app generates mock credentials when prompted by a test-env verification URL.

Mock credentials only work against test flows. They are cryptographically distinct from real credentials and will never verify against a live flow.

## When all else fails

Email support@self.xyz with:

* The relevant request ID (`X-Request-Id` from the API response, or `svix-id` from a webhook delivery).
* Your org ID (visible in the dashboard URL).
* What you expected vs. what happened.
* Approximate timestamps.

We can find a request by ID in seconds; without one, the investigation is much slower.

## Related

* [SDK error handling](../sdk/error-handling.md).
* [Webhook best practices](../webhooks/best-practices.md).
* [Test vs. live](../flows/test-vs-live.md).
