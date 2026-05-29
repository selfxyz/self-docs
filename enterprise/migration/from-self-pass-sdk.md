# From the open-source self-pass SDK

If you've integrated [`@selfxyz/qrcode`](https://docs.self.xyz/self-pass) (the open-source frontend SDK) and `@selfxyz/core` (the backend verifier), here's how it maps to Enterprise.

## Why migrate

The open-source SDK requires you to:

* Stand up an off-chain verifier yourself (or run it in-process and pay the cold-start cost).
* Manage `ConfigStore` for predicate configurations.
* Build your own webhook delivery, retry, and signature scheme.
* Write your own audit log.
* Bill yourself. You can't, really. It's free, with no SLA.

Enterprise replaces all of that with a managed service. You keep your frontend integration largely identical; the backend collapses.

If you're happy running your own verifier and don't need SLAs, billing, or a dashboard, stay on the open-source SDK. Otherwise, read on.

## Concept mapping

| Open-source self-pass | Self Enterprise |
| --- | --- |
| `SelfAppBuilder` (frontend) | Same, but the `endpoint` becomes our hosted verifier; you don't host it. |
| `SelfBackendVerifier` | Replaced by our edge verifier. You don't run this anymore. |
| `ConfigStore` (your impl of `IConfigStorage`) | Replaced by the dashboard's flow configuration. |
| Predicate config object | A **flow** in the dashboard. `flowId` replaces inline config. |
| Per-app secrets (verifier key, etc.) | A Bearer API key (`sk_live_...`). |
| Custom webhook code | A webhook subscription in the dashboard, verified via `SelfWebhooks.verify(...)`. |

## Step-by-step

### 1. Map your existing predicate config to a flow

Today (open-source):

```ts
const verifier = new SelfBackendVerifier({
  scope: 'my-app',
  endpoint: 'https://myapp.com/verify',
  configStore: new InMemoryConfigStorage({
    age_gte: 18,
    nationality_not_in: ['US'],
    ofac: true,
  }),
});
```

With Enterprise:

1. In the dashboard, create a Self Pass flow.
2. Configure the same rules: `age_gte: 18`, `nationality_not_in: ["US"]`, `ofac_clear: true`.
3. Publish. Copy the `flowId`.

### 2. Replace `SelfBackendVerifier` with `SelfClient`

Today:

```ts
import { SelfBackendVerifier } from '@selfxyz/core';

const verifier = new SelfBackendVerifier({ /* ... */ });

app.post('/verify', async (req, res) => {
  const result = await verifier.verify(req.body.attestationId, req.body.proof, req.body.publicSignals, req.body.userContextData);
  // ...
});
```

With Enterprise:

```ts
import { SelfClient } from '@selfxyz/enterprise-sdk';

const self = new SelfClient({ apiKey: process.env.SELF_API_KEY! });

// When the user wants to verify:
app.post('/start-verification', async (req, res) => {
  const session = await self.sessions.create({
    flowId: process.env.SELF_FLOW_ID!,
    externalUuid: req.user.id,
  });
  res.json({ verificationUrl: session.verificationUrl });
});

// You no longer need a /verify endpoint, we deliver the result via webhook.
```

### 3. Replace inline verification with a webhook handler

Today: your `/verify` endpoint runs the Groth16 verifier and acts on the result inline.

With Enterprise: subscribe to `verification.completed` in the dashboard, and handle it like this:

```ts
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

app.post('/webhooks/self', express.raw({ type: 'application/json' }), (req, res) => {
  const event = SelfWebhooks.verify(req.body, req.headers, process.env.SELF_WEBHOOK_SECRET!);

  if (event.type === 'verification.completed' && event.status === 'valid') {
    grantAccess(event.external_uuid, event.proof_attributes);
  }
  res.status(200).end();
});
```

The `external_uuid` is what you passed when creating the session (typically your user ID).

### 4. Update your frontend QR code component (if you use one)

If you were rendering your own QR code via `@selfxyz/qrcode`, the simplest path is to **redirect to `session.verificationUrl`** instead. Our hosted page renders the QR code and handles deeplinks.

If you'd rather keep rendering the QR inline:

* Call `self.sessions.create(...)` from your backend.
* Pass `session.verificationUrl` to your QR component as the data.

The user-facing UX is unchanged.

### 5. Delete the old code paths

* Remove `@selfxyz/core` and `@selfxyz/qrcode` (unless you're still rendering the QR yourself).
* Remove your `ConfigStore` implementation.
* Remove your `/verify` route (Enterprise delivers via webhook).
* Remove any code that loaded verifier circuit files at boot.

## What stays the same

* The **Self mobile app** is unchanged for your users.
* The **disclosures** are the same, `age_gte`, `nationality_not_in`, `ofac_clear`, etc.
* The **proof system** is the same Groth16 under the hood.
* Your **frontend** can continue rendering its own QR if you want, just pointed at our session URL.

## What's better

* No verifier infrastructure to run.
* Audit log out of the box.
* Webhook delivery with retries and replay.
* A dashboard to change rules without a redeploy.
* Per-flow versioning so you can audit "what was this user verified against in March".

## What's different

* You pay per verification (see [Plans](../billing/plans.md)).
* Your service no longer holds the raw proof, we verify it, you get the attributes. (You can still pull the raw proof from the `verification.completed` event if you really need it.)

## Rolling out

A safe rollout pattern:

1. Build the Enterprise integration in a feature flag, behind `enterprise: false`.
2. Configure a test-environment flow + webhook subscription.
3. QA against mock passports.
4. Move staging to Enterprise (`enterprise: true` for non-prod).
5. Cutover prod with a percentage rollout (`enterprise: true` for 10%, 50%, 100%).
6. Decommission your verifier infra after a week of clean prod.

## Need help?

Email migrating@self.xyz, or your CSM if you're on Enterprise.
