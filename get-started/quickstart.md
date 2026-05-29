# Quickstart

End-to-end: from sign-up to a verified user, in about ten minutes.

## Prerequisites

* A Self.xyz dashboard account ([sign up](https://dashboard.self.xyz)).
* Node 18+ (for the SDK).
* A way to receive a webhook locally, [ngrok](https://ngrok.com), Cloudflare Tunnel, or any public HTTPS endpoint.

## 1. Create your organization

After sign-up you'll be prompted to create an organization. An org owns flows, API keys, webhook subscriptions, and the billing relationship. You can invite teammates from **Settings → People** once it's set up.

## 2. Configure a Self Pass flow

From the dashboard home, choose **Self Pass** and click **Create flow**.

A flow has three parts:

* **Rules**: the disclosures you want (e.g. `age >= 18`, `nationality not in ['US', 'CN']`, `not on OFAC list`).
* **Documents**: which credentials are acceptable (passport, Aadhaar, KYC attestation).
* **Settings**: things like success/failure redirect URLs.

Fill them in, then click **Publish version**. Your flow now has a published `flowId`, copy it.

> 📸 _**Screenshot:** Published flow card showing the `flowId` and a "Copy" button._

> **Test vs. live:** A flow published in your test environment only accepts mock passports and never bills credits. See [Test vs. live](../flows/test-vs-live.md).

## 3. Create an API key

**Settings → API keys → Create key**. Choose `test` while you're integrating. The key (`sk_test_...`) is shown once, store it as `SELF_API_KEY` in your backend's secret manager.

> 📸 _**Screenshot:** API key creation flow, showing the env selector (test/live) and the masked key after creation._

## 4. Install the SDK

```bash
npm install @selfxyz/enterprise-sdk
```

## 5. Create a verification session

```ts
import { SelfClient } from '@selfxyz/enterprise-sdk';

const self = new SelfClient({ apiKey: process.env.SELF_API_KEY! });

const session = await self.sessions.create({
  flowId: '<paste flowId from step 2>',
  externalUuid: 'user_42',           // your stable identifier for the user
});

console.log(session.verificationUrl);  // hand this to the user
```

The user opens `verificationUrl` in their Self app, produces a proof, and the app submits it back to us.

## 6. Subscribe to webhooks

**Settings → Webhooks → Add endpoint**. Paste your public URL (e.g. `https://<your-tunnel>/webhooks/self`) and subscribe to `verification.completed`.

The dashboard returns a signing secret (`whsec_...`). Store it as `SELF_WEBHOOK_SECRET`.

## 7. Verify webhook deliveries

```ts
import express from 'express';
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

const app = express();

app.post(
  '/webhooks/self',
  express.raw({ type: 'application/json' }),  // raw body is required for signature verification
  (req, res) => {
    try {
      const event = SelfWebhooks.verify(
        req.body,                                            // Buffer
        req.headers as Record<string, string>,
        process.env.SELF_WEBHOOK_SECRET!,
      );

      if (event.type === 'verification.completed') {
        // event.verification_id, event.external_uuid, event.proof_attributes
        console.log('verified:', event);
      }

      res.status(200).end();
    } catch (err) {
      res.status(400).end();
    }
  },
);
```

## 8. Test the loop

Open the `verificationUrl` from step 5, run through with a mock passport, and watch your webhook handler fire. The dashboard's **Activity log** tab on the flow shows the verification end-to-end.

> 📸 _**Screenshot:** Hosted verification page (with QR code) next to the Activity log showing a completed session._

## What's next

* [Configure a product](../dashboard/configure-a-product.md): go beyond defaults.
* [Event catalog](../webhooks/events.md): every webhook type and its payload.
* [SDK reference](../sdk/nodejs.md): every SDK method, with types.
* [Error handling](../sdk/error-handling.md): what each error means and how to handle it.
