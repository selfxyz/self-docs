# Node.js / TypeScript SDK

`@selfxyz/enterprise-sdk` is the official Node/TS client for Self Enterprise. It wraps session creation, retrieval, and webhook signature verification with typed payloads.

## Install

{% tabs %}
{% tab title="npm" %}
```bash
npm install @selfxyz/enterprise-sdk
```
{% endtab %}

{% tab title="pnpm" %}
```bash
pnpm add @selfxyz/enterprise-sdk
```
{% endtab %}

{% tab title="yarn" %}
```bash
yarn add @selfxyz/enterprise-sdk
```
{% endtab %}
{% endtabs %}

Node 20+. ESM-only.

## Initialize

```ts
import { SelfClient } from '@selfxyz/enterprise-sdk';

const self = new SelfClient({
  apiKey: process.env.SELF_API_KEY!,        // sk_test_... or sk_live_...
});
```

The API key is the only option you need to pass. Whether the client talks to test or live is determined by the key prefix (`sk_test_` or `sk_live_`).

## Sessions

### Create

Get the `flowId` by publishing a configuration in the dashboard, it's shown on the product's **Test** and **Live** tabs. See [Configure a product](../dashboard/configure-a-product.md).

```ts
const session = await self.sessions.create({
  flowId: '9c0b4f1c-1d6c-4f1b-a8c4-9f0fa0a8d9e2',
  externalUuid: 'a1b2c3d4-5678-4e9a-b012-3456789abcde',  // a UUID, your stable id for the user
  // optional:
  expiresInSeconds: 3600,
  metadata: { campaign: 'winter-2026' },
  successUrl: 'https://app.example.com/verified',
  failureUrl: 'https://app.example.com/failed',
});

session.verificationUrl;  // give to the user
session.id;               // store on your side
session.expiresAt;        // ISO-8601
```

Returns `Session` (alias of `CreateSessionResponse`).

The most important fields:

| Field | Notes |
| --- | --- |
| `id` | Persistent session ID. Use this when calling `sessions.get(...)` or matching against webhook events. |
| `verificationUrl` | The URL to hand to the user. They open it in their Self app. |
| `expiresAt` | ISO-8601 timestamp. After this the session cannot be completed. |
| `flowVersionId` | Immutable pin to the flow version this session ran against. Publishing a new flow version doesn't disturb in-flight sessions. |

### Get

```ts
const detail = await self.sessions.get('7f3b2a1e-9c4d-4b2a-8e1f-2c6d5a4b3c2d');  // the session id

detail.status;             // 'pending' | 'valid' | 'invalid' | 'error' | 'expired'
detail.proofAttributes;    // disclosed predicates, e.g. { age_gte_18: true }
detail.storage.state;      // 'pending' | 'committed' | 'failed'
```

Returns `SessionDetail` (exported from the SDK, an alias of `SessionDetailResponse`):

```ts
interface SessionDetail {
  id: string;
  status: 'pending' | 'valid' | 'invalid' | 'error' | 'expired';
  createdAt: string;                                  // ISO-8601
  completedAt: string | null;                         // ISO-8601, null until terminal
  expiresAt: string;                                  // ISO-8601
  flowVersionId: string;                              // the pinned flow version
  externalUuid: string;                               // your identifier from create()
  metadata: Record<string, unknown> | null;           // what you passed to create()
  predicatesConfig: Record<string, unknown> | null;   // the rules this ran against
  proofAttributes: Record<string, unknown> | null;    // disclosed results (null until valid)
  storage: {
    state: 'pending' | 'committed' | 'failed';
    uri: string | null;
    credentialId: string | null;
  };
}
```

## Types

The SDK re-exports the canonical Zod schemas and inferred types from `@self/schemas`:

```ts
import type {
  CreateSessionInput,           // what you pass to .create()
  Session,                      // what .create() returns
  SessionDetail,                // what .get() returns
  WebhookEvent,                 // discriminated-union of webhook event payloads
  VerificationCompletedPayload, // the verification.completed payload
} from '@selfxyz/enterprise-sdk';
```

You can also import the schemas themselves for runtime validation:

```ts
import { createSessionBody, webhookEvent } from '@selfxyz/enterprise-sdk';

const parsed = createSessionBody.parse(input);
```

## Webhook verification

See [Verify webhooks](verify-webhooks.md).

## Error handling

```ts
import { SelfClient, SelfApiError } from '@selfxyz/enterprise-sdk';

try {
  await self.sessions.create({ flowId, externalUuid });
} catch (err) {
  if (err instanceof SelfApiError) {
    err.statusCode;    // 400 | 401 | 402 | 403 | 404 | 409 | 429 | 5xx
    err.code;          // 'validation_failed' | 'unauthenticated' | 'not_found' | ...
    err.message;       // human-readable
    err.details;       // optional extra context (Record<string, unknown>)
    err.requestId;     // X-Request-Id, quote it to support
  }
}
```

Bad arguments (for example a `flowId` or `externalUuid` that isn't a UUID) throw `SelfValidationError` before any request is sent. The SDK doesn't retry, handle transient `429` and `5xx` responses yourself (back off and retry). See [Error handling](error-handling.md) for the full code catalog.
