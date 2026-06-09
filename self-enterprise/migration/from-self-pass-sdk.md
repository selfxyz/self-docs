---
icon: arrow-right-arrow-left
description: Move from the open-source self-pass SDK to the managed Enterprise platform.
---

# From the open-source SDK

If you've integrated the open-source [Self Pass](../../self-pass/README.md) frontend SDK (`@selfxyz/qrcode`) and backend verifier (`@selfxyz/core`), here's how it maps to Self Enterprise.

## Why migrate

The open-source SDK requires you to:

* Stand up an off-chain verifier yourself (or run it in-process and pay the cold-start cost).
* Manage a `ConfigStore` for predicate configurations.
* Build your own webhook delivery, retry, and signature scheme.
* Write your own audit log.
* Operate it all with no SLA.

Enterprise replaces all of that with a managed service. Your frontend integration stays largely identical; the backend collapses.

{% hint style="info" %}
If you're happy running your own verifier and don't need SLAs, billing, or a dashboard, stay on the open-source SDK, it's fully supported. Otherwise, read on.
{% endhint %}

## Concept mapping

| Open-source self-pass | Self Enterprise |
| --- | --- |
| `SelfAppBuilder` (frontend) | Same idea, but the `endpoint` becomes our hosted verifier, you don't host it. |
| `SelfBackendVerifier` | Replaced by our managed verifier. You don't run this anymore. |
| `ConfigStore` (your `IConfigStorage` impl) | Replaced by the dashboard's [flow configuration](../dashboard/configure-a-product.md). |
| Predicate config object | A **flow** in the dashboard. `flowId` replaces inline config. |
| Per-app verifier secrets | A Bearer API key (`sk_live_…`). |
| Custom webhook code | A webhook endpoint you register, verified via `SelfWebhooks.verify(...)`. |

## Step-by-step

### 1. Map your predicate config to a flow

Today (open-source):

```ts
const verifier = new SelfBackendVerifier({
  scope: 'my-app',
  endpoint: 'https://myapp.com/verify',
  configStore: new InMemoryConfigStorage({
    minimumAge: 18,
    excludedCountries: ['USA'],
    ofac: true,
  }),
});
```

With Enterprise:

1. In the dashboard, [create a flow](../dashboard/configure-a-product.md).
2. Configure the same rules: minimum age 18, exclude `USA`, OFAC on.
3. Publish it. Copy the `flowId` from the **Test** or **Live** tab.

### 2. Replace `SelfBackendVerifier` with `SelfClient`

Today:

```ts
import { SelfBackendVerifier } from '@selfxyz/core';

const verifier = new SelfBackendVerifier({ /* ... */ });

app.post('/verify', async (req, res) => {
  const result = await verifier.verify(
    req.body.attestationId,
    req.body.proof,
    req.body.publicSignals,
    req.body.userContextData,
  );
  // ...act on result
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

Today your `/verify` endpoint runs the Groth16 verifier and acts on the result inline.

With Enterprise, register a webhook endpoint in the dashboard (it receives all events) and handle `verification.completed`:

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

`external_uuid` is what you passed when creating the session (typically your user ID). See [Verify webhooks](../sdk/verify-webhooks.md).

### 4. Drop your frontend QR component

With Enterprise you don't render your own QR. Wherever you used `@selfxyz/qrcode`, call `self.sessions.create(...)` on your backend and **redirect the user to `session.verificationUrl`** instead; Self's hosted page renders the QR and handles deeplinks. Then remove `@selfxyz/qrcode`.

### 5. Delete the old code paths

* Remove `@selfxyz/core` and `@selfxyz/qrcode`.
* Remove your `ConfigStore` implementation.
* Remove your `/verify` route, Enterprise delivers via webhook.
* Remove any code that loaded verifier circuit files at boot.

## What stays the same

* The **Self mobile app** is unchanged for your users.
* The **disclosures** are the same, age, nationality, OFAC, etc. See [Disclosures](../flows/disclosures.md).
* The **proof system** is the same under the hood.
* Your **frontend** gets simpler: send the user to the hosted `verificationUrl`, no QR rendering required.

## What's better

* No verifier infrastructure to run.
* [Audit log](../dashboard/activity-log.md) out of the box.
* [Webhook delivery](../webhooks/overview.md) with automatic retries.
* A [dashboard](../dashboard/overview.md) to change rules without a redeploy.
* [Per-flow versioning](../flows/anatomy.md#versions), audit exactly what a user was verified against last March.

## What's different

* You pay per verification. See [Plans](../billing/plans.md).
* Your service no longer holds the raw proof, we verify it and hand you the attributes. (You can still read the raw proof from the `verification.completed` event if you truly need it.)

## Rolling out safely

1. Build the Enterprise integration behind a feature flag (`enterprise: false`).
2. Configure a **test**-environment flow + webhook endpoint.
3. QA against [mock passports](../guides/using-mock-passports.md).
4. Move staging to Enterprise.
5. Cut over prod with a percentage rollout (10% → 50% → 100%).
6. Decommission your verifier infra after a week of clean prod traffic.

## Need help?

Email support@self.xyz, or your CSM if you're on an Enterprise plan.
