# SDK API reference

The public surface of `@selfxyz/enterprise-sdk` you use to create sessions and verify webhooks. For a guided walkthrough, start with the [SDK overview](nodejs.md).

{% hint style="info" %}
The Enterprise SDK is **backend-only**, and there's no frontend SDK or QR code to render yourself. Your frontend just redirects the user to the `verificationUrl` a session returns; Self hosts the QR and deeplink. Everything below runs on your server.
{% endhint %}

## Exports

```ts
import {
  SelfClient,               // the API client
  SelfWebhooks,             // webhook signature verification
  SelfApiError,             // thrown on a non-2xx API response
  SelfValidationError,      // thrown on invalid input / unknown payload
  WebhookVerificationError, // thrown on a bad webhook signature
} from '@selfxyz/enterprise-sdk';

import type {
  CreateSessionInput,                  // argument to sessions.create()
  Session,                             // return of sessions.create()
  SessionDetail,                       // return of sessions.get()
  WebhookEvent,                        // discriminated union of webhook events
  VerificationCompletedPayload,        // the verification.completed payload
} from '@selfxyz/enterprise-sdk';
```

The Zod schemas behind those types (`createSessionBody`, `sessionDetailResponse`, `webhookEvent`) are also exported as values if you want runtime validation.

---

## `SelfClient`

```ts
new SelfClient({ apiKey: string })
```

The API client. `apiKey` is your `sk_test_…` or `sk_live_…` key; the environment is determined by its prefix. Throws if `apiKey` is missing.

### `sessions.create(input)`

```ts
sessions.create(input: CreateSessionInput): Promise<Session>
```

Creates a verification session against the published flow and returns a URL to send the user to.

**`CreateSessionInput`**

| Field | Type | Required | Notes |
| --- | --- | :---: | --- |
| `flowId` | string (UUID) | ✅ | The flow to verify against. |
| `externalUuid` | string (UUID) | ✅ | Your stable identifier for the user. Must be a UUID. |
| `expiresInSeconds` | number | No | How long the session stays openable. Integer, 60 to 86400. Default 3600. |
| `metadata` | object | No | Arbitrary JSON attached to the session. Max 4 KB serialized. |
| `successUrl` | string (URL) | No | Where the hosted page sends the user on success. |
| `failureUrl` | string (URL) | No | Where the hosted page sends the user on failure. |

**`Session`** (the return)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | The session ID. Match it against webhook `verification_id`. |
| `externalUuid` | string | Echoes your input. |
| `status` | `'pending'` | Always `pending` at creation. |
| `flowVersionId` | string | The pinned flow version. |
| `verificationToken` | string | Opaque token (`verify_<env>_…`). |
| `verificationUrl` | string | The URL to hand to the user. |
| `createdAt` | string | ISO-8601. |
| `expiresAt` | string | ISO-8601. |
| `completedAt` | null | Always `null` at creation. |

Throws [`SelfApiError`](#selfapierror) on a non-2xx response.

### `sessions.get(id)`

```ts
sessions.get(id: string): Promise<SessionDetail>
```

Reads a session's current state.

**`SessionDetail`** (the return)

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | The session ID. |
| `status` | `'pending' \| 'valid' \| 'invalid' \| 'error' \| 'expired'` | Current state. |
| `createdAt` | string | ISO-8601. |
| `completedAt` | string \| null | ISO-8601, `null` until terminal. |
| `expiresAt` | string | ISO-8601. |
| `flowVersionId` | string | The pinned flow version. |
| `externalUuid` | string | Your identifier from create. |
| `metadata` | object \| null | What you passed to create. |
| `predicatesConfig` | object \| null | The rules this ran against. |
| `proofAttributes` | object \| null | Disclosed results, `null` until valid. |
| `storage` | object | `{ state: 'pending' \| 'committed' \| 'failed'; uri: string \| null; credentialId: string \| null }`. |

Throws [`SelfApiError`](#selfapierror) (for example `404` if the ID is unknown).

---

## `SelfWebhooks`

### `SelfWebhooks.verify(payload, headers, secret)`

```ts
SelfWebhooks.verify(
  payload: string | Buffer,
  headers: Record<string, string>,
  secret: string,
): WebhookEvent
```

Verifies a webhook signature and returns the typed, parsed event.

| Parameter | Notes |
| --- | --- |
| `payload` | The **raw** request body (string or Buffer), not a parsed object. |
| `headers` | Must include `svix-id`, `svix-timestamp`, `svix-signature`. |
| `secret` | The endpoint's signing secret (`whsec_…`). |

Throws [`WebhookVerificationError`](#webhookverificationerror) if the signature is invalid or the timestamp is stale, and [`SelfValidationError`](#selfvalidationerror) if the body doesn't match any known event shape (usually an out-of-date SDK).

**`WebhookEvent`** is a discriminated union on `type`. The event delivered to your endpoints is `verification.completed` (`VerificationCompletedPayload`):

```ts
if (event.type === 'verification.completed' && event.status === 'valid') {
  // event is VerificationCompletedPayload here
  await markUserVerified(event.external_uuid, event.proof_attributes);
}
```

See the [event catalog](../webhooks/events.md) for the payload's fields.

---

## Errors

### `SelfApiError`

Thrown when the API returns a non-2xx response.

| Property | Type | Notes |
| --- | --- | --- |
| `statusCode` | number | HTTP status. |
| `code` | string | Stable machine-readable code (`validation_failed`, `not_found`, ...). |
| `message` | string | Human-readable. |
| `details` | `Record<string, unknown>` \| undefined | Extra context (for example validation `issues`). |
| `requestId` | string \| undefined | The `X-Request-Id` response header, when present. Quote it to support. |

See [Error handling](error-handling.md) for the code catalog.

### `SelfValidationError`

Thrown when arguments fail schema validation **before** a request is sent (for example a `flowId` or `externalUuid` that isn't a UUID), and by `SelfWebhooks.verify(...)` when a verified payload doesn't match any known event shape.

| Property | Type | Notes |
| --- | --- | --- |
| `issues` | `ZodIssue[]` | Which fields failed and why. |
| `message` | string | Human-readable summary. |

### `WebhookVerificationError`

Thrown by `SelfWebhooks.verify(...)` on a bad signature, stale timestamp, or missing headers. Respond `400`, a bad signature won't pass on retry.

## Related

* [SDK overview](nodejs.md): the guided version.
* [Verify webhooks](verify-webhooks.md): framework wiring.
* [Error handling](error-handling.md): the error catalog.
