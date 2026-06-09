# Verify webhooks

The SDK ships a `SelfWebhooks.verify(...)` helper that checks the signature and returns a typed event payload.

## Setup

You need:

* The **raw** request body (string or Buffer). Not the JSON-parsed object, signature verification operates on the byte string.
* The request headers from the delivery. Pass them through as-is; they carry the signature the SDK checks.
* The signing secret (`whsec_...`) for the webhook endpoint. You get it once, when you [add the endpoint](../dashboard/webhooks.md).

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
  const raw = await c.req.text();                          // raw body as string
  const headers = Object.fromEntries(c.req.raw.headers);   // pass all headers through

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
  const headers = Object.fromEntries(req.headers);   // pass all headers through

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
