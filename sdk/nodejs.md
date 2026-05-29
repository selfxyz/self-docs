# Node.js / TypeScript SDK

`@selfxyz/enterprise-sdk` is the official Node/TS client for the Enterprise API. It wraps session creation, retrieval, and webhook signature verification with typed payloads.

## Install

```bash
npm install @selfxyz/enterprise-sdk
# or
pnpm add @selfxyz/enterprise-sdk
# or
yarn add @selfxyz/enterprise-sdk
```

Node 18+. ESM-only.

## Initialize

```ts
import { SelfClient } from '@selfxyz/enterprise-sdk';

const self = new SelfClient({
  apiKey: process.env.SELF_API_KEY!,        // sk_test_... or sk_live_...
});
```

### Options

```ts
interface SelfClientOptions {
  /** Bearer API key. Required. */
  apiKey: string;

  /** Environment hint. The actual environment is embedded in the API key. */
  environment?: 'test' | 'live';

  /** Override the base URL (default: https://api.self.xyz). */
  baseUrl?: string;
}
```

`environment` is a hint for your own code clarity, the live behavior is determined by the key prefix. `baseUrl` is useful for staging or self-hosted edges.

## Sessions

### Create

```ts
const session = await self.sessions.create({
  flowId: '9c0b4f1c-1d6c-4f1b-a8c4-9f0fa0a8d9e2',
  externalUuid: 'user_42',
  // optional:
  expiresInSeconds: 3600,
  metadata: { campaign: 'winter-2026' },
  successUrl: 'https://app.example.com/verified?u=42',
  failureUrl: 'https://app.example.com/failed?u=42',
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
const detail = await self.sessions.get('ver_01HXYZ...');

detail.status;             // 'pending' | 'valid' | 'invalid' | 'error' | 'expired'
detail.proofAttributes;    // disclosed predicates, e.g. { age_gte_18: true }
detail.storage.state;      // 'pending' | 'committed' | 'failed'
```

Returns `SessionDetail` (alias of `SessionDetailResponse`).

## Types

The SDK re-exports the canonical Zod schemas and inferred types from `@self/schemas`:

```ts
import type {
  CreateSessionInput,           // what you pass to .create()
  Session,                      // what .create() returns
  SessionDetail,                // what .get() returns
  WebhookEvent,                 // discriminated-union of all event payloads
  VerificationCompletedPayload,
  VerificationStorageCommittedPayload,
  VerificationStorageFailedPayload,
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
    err.status;        // 400 | 401 | 402 | 403 | 404 | 429 | 5xx
    err.code;          // 'validation_failed' | 'unauthorized' | ...
    err.message;       // human-readable
    err.details;       // optional extra context
  }
}
```

See [Error handling](error-handling.md) for the full catalog and retry guidance.

## Compatibility

The SDK uses `.passthrough()` on webhook event schemas, so adding new fields on the server side is non-breaking. New event types or new request/response fields ship in a minor version; renames or removals ship in a major version.

The package is pre-1.0 (`0.x`). Minor versions may contain breaking changes until 1.0. Pin to an exact version in production.
