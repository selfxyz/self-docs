---
icon: life-ring
---

# Troubleshooting

Things that commonly go wrong, and how to unstick them. Errors from the SDK arrive as `SelfApiError` (`err.statusCode`, `err.code`); the sections below are keyed by those.

If none of these apply, email support@self.xyz with your org ID (from the dashboard URL) and, for a webhook issue, the `verification_id` from the event payload.

## Authentication

### `401 unauthenticated`

The API key is missing, malformed, or revoked.

* Confirm the `apiKey` you pass to `SelfClient` is set and is the full value (it starts with `sk_test_` or `sk_live_`).
* If it works locally but 401s in deploy, you're probably reading the wrong env var. If it worked before and suddenly 401s, the key was likely revoked.
* The full key is shown once on creation. If you've lost it, generate a new one under **Developer → API keys** and revoke the old one.

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

### `404 not_found`

The `flowId` (or session ID) is wrong, points at a draft (not yet published), or the flow is archived. A flow that exists but has no published version returns `409 conflict` instead.

* Open the flow in the dashboard. If you don't see a "Published" badge, click **Publish version** (otherwise you'll get `409 conflict`).
* If the flow was deleted, recreate it; the ID is gone.

### `402`, insufficient credits

Your org's credit balance is too low to cover this session's cost, and your [credit gate](../billing/credits-and-usage.md#insufficient-credits) is set to `hard`. (The error envelope carries `code: "unauthenticated"` with HTTP `402`, branch on the status, not the code.)

* Check your balance on the **Settings → Usage & Billing** tab.
* On Free, the grant resets at the start of each cycle. On Starter, upgrade your plan (or talk to sales about Enterprise) for more capacity.
* Watch the credits meter on that tab so this doesn't surprise you.

### `429 rate_limited`

Per-API-key rate limit exceeded. Honor the `Retry-After` header.

* If your traffic legitimately exceeds the limit, contact sales@self.xyz about higher limits or move to Enterprise.
* If a single key is doing all the work, split traffic across multiple keys, limits are per-key.
* The SDK doesn't retry, your code should back off and retry on `429`.

### `5xx` errors

Transient. Retry with exponential backoff. If it's persistent (more than 30 seconds), check [status.self.xyz](https://status.self.xyz) and contact support with the request ID.

## Webhooks

### Webhook handler never receives events

* Confirm the endpoint is registered under **Developer → Webhooks** for the **same environment** as the key creating sessions (test vs live).
* Confirm your endpoint is reachable from the public internet (not behind a VPN, no firewall blocking POST).
* If using a tunnel (ngrok, Cloudflare Tunnel), confirm the tunnel is still up.
* The delivered event is `verification.completed`; branch on `event.type` so a future event type doesn't surprise your handler.

### Webhook deliveries fail with `400`

Your handler is rejecting the delivery. Most likely:

* **Signature verification failed.** Check that:
  * You're using the *raw* body, not a JSON-parsed object.
  * The signing secret matches the dashboard's `whsec_...` for this endpoint.
  * You're not rewriting headers between proxy and handler.
* **Body schema mismatch.** Update to the latest SDK version, we may have added an event type your version doesn't know.

### Webhook deliveries fail with `5xx`

Your handler is erroring before completing. Find the matching invocation in your logs by the `verification_id` from the event payload. Self retries `5xx`, so once you fix the handler the next retry should land.

### Duplicate events

Expected. We deliver at-least-once. See [Best practices](../webhooks/best-practices.md#1-make-handlers-idempotent) for dedup patterns.

## SDK

### `Cannot find module '@selfxyz/enterprise-sdk'`

The package is ESM-only and requires Node 20+.

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

In a **test** environment (sessions created with an `sk_test_` key) you verify with a mock passport from the Self app instead of a real document. See [Using mock passports](../guides/using-mock-passports.md) for the setup. Mock credentials are cryptographically distinct from real ones and never verify against a live flow.

## When all else fails

Email support@self.xyz with:

* Your Email.
* The `err.code` and `err.message` from the `SelfApiError`, or the `verification_id` from the event for a webhook issue.
* What you expected vs. what happened, with approximate timestamps.

The more of that you include, the faster we can trace it.

## Related

* [SDK error handling](../sdk/error-handling.md).
* [Webhook best practices](../webhooks/best-practices.md).
* [Test vs. live](../flows/test-vs-live.md).
