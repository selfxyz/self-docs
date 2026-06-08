# Verify webhooks

The SDK ships a `SelfWebhooks.verify(...)` helper that checks the signature and returns a typed event payload.

## Setup

You need:

* The **raw** request body (string or Buffer). Not the JSON-parsed object, signature verification operates on the byte string.
* The request headers, at minimum `svix-id`, `svix-timestamp`, `svix-signature`.
* The signing secret (`whsec_...`) for the webhook endpoint. You get it when you [add the endpoint](../dashboard/webhooks.md) (revealed once); note the endpoint must return `2xx` to be saved.

## Express

```ts
import express from 'express';
import { SelfWebhooks, WebhookVerificationError } from '@selfxyz/enterprise-sdk';

const app = express();

app.post(
  '/webhooks/self',
  express.raw({ type: 'application/json' }),    // keep the raw bytes
  (req, res) => {
    try {
      const event = SelfWebhooks.verify(
        req.body,                                  // Buffer
        req.headers as Record<string, string>,
        process.env.SELF_WEBHOOK_SECRET!,
      );

      if (event.type === 'verification.completed') {
        // event.verification_id, event.external_uuid, event.proof_attributes, event.status
      }

      res.status(200).end();
    } catch (err) {
      if (err instanceof WebhookVerificationError) {
        res.status(400).end();
      } else {
        // Unknown payload shape (SelfValidationError) or server bug, log and 5xx so we retry.
        res.status(500).end();
      }
    }
  },
);
```

## Hono

```ts
import { Hono } from 'hono';
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

const app = new Hono();

app.post('/webhooks/self', async (c) => {
  const raw = await c.req.text();                 // raw body as string
  const headers: Record<string, string> = {
    'svix-id': c.req.header('svix-id') ?? '',
    'svix-timestamp': c.req.header('svix-timestamp') ?? '',
    'svix-signature': c.req.header('svix-signature') ?? '',
  };

  try {
    const event = SelfWebhooks.verify(raw, headers, process.env.SELF_WEBHOOK_SECRET!);
    // ...handle event
    return c.text('ok');
  } catch {
    return c.text('bad signature', 400);
  }
});
```

## Next.js (App Router)

```ts
// app/api/webhooks/self/route.ts
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';
import { NextRequest } from 'next/server';

export async function POST(req: NextRequest) {
  const raw = await req.text();
  const headers: Record<string, string> = {
    'svix-id': req.headers.get('svix-id') ?? '',
    'svix-timestamp': req.headers.get('svix-timestamp') ?? '',
    'svix-signature': req.headers.get('svix-signature') ?? '',
  };

  try {
    const event = SelfWebhooks.verify(raw, headers, process.env.SELF_WEBHOOK_SECRET!);
    // ...
    return new Response('ok', { status: 200 });
  } catch {
    return new Response('bad signature', { status: 400 });
  }
}
```

## Type narrowing

`event` is a discriminated union on `event.type`. TypeScript narrows automatically:

```ts
if (event.type === 'verification.completed') {
  event.status;              // 'valid' | 'invalid' | 'error' | 'expired'
  event.proof_attributes;    // disclosed predicates
  // event.storage_uri is also available
}
```

## Common mistakes

* **Parsing the body before verification.** If you `app.use(express.json())`, the raw body is gone by the time the route runs and verification will fail. Use `express.raw(...)` for the webhook path.
* **Using the wrong secret.** Each endpoint has its own `whsec_...`. If you have multiple endpoints registered (e.g. staging + prod), don't share secrets across them.
* **Trusting the payload without verifying.** Always call `SelfWebhooks.verify(...)` first. Don't parse `event.type` until verification succeeds.

## Redeliveries

The same event can arrive more than once (a retry after a failed delivery, for example). The body and signature are valid each time, so verification succeeds normally. Make your handler idempotent, dedupe on the event's `verification_id`.

See [Best practices](../webhooks/best-practices.md) for idempotency patterns.
