# SDK error handling

The SDK throws two error classes. Catch them by type.

## `SelfApiError`

Thrown when the Enterprise API returns a non-2xx response.

```ts
import { SelfApiError } from '@selfxyz/enterprise-sdk';

try {
  await self.sessions.create({ flowId, externalUuid });
} catch (err) {
  if (err instanceof SelfApiError) {
    err.status;     // number (HTTP status)
    err.code;       // string ('validation_failed', 'flow_not_found', ...)
    err.message;    // string (human-readable)
    err.details;    // Record<string, unknown> | undefined
  } else {
    throw err;      // network error, unexpected
  }
}
```

### Switching on `code`

```ts
import { SelfApiError } from '@selfxyz/enterprise-sdk';

try {
  await self.sessions.create({ flowId, externalUuid });
} catch (err) {
  if (!(err instanceof SelfApiError)) throw err;

  switch (err.code) {
    case 'validation_failed':
      // err.details.issues is a Zod-issue array
      return badRequest(err.details);

    case 'flow_not_found':
      return notFound('That flow no longer exists');

    case 'insufficient_credits':
      // Trigger top-up flow / alert ops
      return paymentRequired();

    case 'rate_limited':
      // SDK already retried; we're out of budget
      return tryLater();

    default:
      return serverError(err);
  }
}
```

### Error codes

| `code` | `status` | When | What to do |
| --- | --- | --- | --- |
| `validation_failed` | 400 | Request body didn't match the schema. `details.issues` lists offending fields. | Fix the request; do not retry. |
| `unauthorized` | 401 | Missing, malformed, or revoked API key. | Check your key. |
| `forbidden` | 403 | Key valid but not allowed for this resource (e.g. test key + live flow, cross-org session). | Use a key in the right environment / org. |
| `flow_not_found` | 404 | `flowId` doesn't exist, isn't published, or is archived. | Verify the ID in the dashboard. |
| `session_not_found` | 404 | `sessionId` doesn't exist or belongs to another org. | Verify the ID. |
| `insufficient_credits` | 402 | Org credit balance too low to cover this session. | Top up; either wait for the next billing period or upgrade plan. |
| `rate_limited` | 429 | Per-key rate limit exceeded. | Honor the `Retry-After` header. The SDK already retries with backoff. |
| `internal_error` | 500 | Server-side bug. | Retry with backoff. If persistent, contact support with the request ID. |
| `service_unavailable` | 503 | Temporary degradation. | Retry with backoff. |

## `WebhookVerificationError`

Thrown by `SelfWebhooks.verify(...)` when the signature doesn't match the body, the timestamp is too old, or required headers are missing.

```ts
import { WebhookVerificationError } from '@selfxyz/enterprise-sdk';

try {
  const event = SelfWebhooks.verify(raw, headers, secret);
} catch (err) {
  if (err instanceof WebhookVerificationError) {
    // Respond 400. We won't retry a 4xx, which is what you want for a bad signature.
  }
}
```

A separate failure mode: if the signature checks out but the body shape doesn't match any known event schema, `verify(...)` throws a Zod `ZodError`. That usually means you're on an old SDK version and the server is sending a newer event type, upgrade the SDK.

## Retrying

The SDK already retries internally on transient errors:

* `429 rate_limited`: exponential backoff honoring `Retry-After`.
* `5xx`: exponential backoff, capped at 5 attempts.

You don't need to wrap calls in your own retry loop. If you do, retry only on these statuses, never on 4xx.

## Logging

`SelfApiError` carries a `requestId` field when the server included `X-Request-Id`. Log it:

```ts
log.error({
  msg: 'self_api_error',
  code: err.code,
  status: err.status,
  requestId: err.requestId,
  details: err.details,
});
```

Support can find a request by `requestId` in seconds; without it, much slower.

## Related

* [Webhook signature verification](../webhooks/signature-verification.md).
* [Best practices](../webhooks/best-practices.md).
