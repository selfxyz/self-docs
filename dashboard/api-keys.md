# API keys

**Settings → API keys**. One screen per environment.

> 📸 _**Screenshot:** API keys list in the dashboard, with a few keys shown and the **Create key** button visible._

## Issuing a key

Click **Create key**. You'll choose:

* **Environment**: `test` or `live`. The choice is baked into the key prefix and the routing. Test keys can only create sessions against test flows; live keys against live flows. There's no cross-environment access.
* **Name**: a human label (e.g. `prod-backend`, `staging`, `local-kartik`).

The key is shown **once**. Copy it into your secret manager (e.g. GCP Secret Manager, AWS Secrets Manager, 1Password). We never display the full key again. From then on, only the masked form (`sk_test_•••8a3f`) is visible.

> 📸 _**Screenshot:** The "key created" modal, showing the full key once with a Copy button._

## Key shape

```
sk_test_<22-char-base62>
sk_live_<22-char-base62>
```

The prefix is intentional and stable; treat it as part of the credential format. Detecting `sk_live_` in commits is a useful pre-push hook.

## Using a key

Hand it to the Enterprise SDK at initialization:

```ts
import { SelfClient } from '@selfxyz/enterprise-sdk';

const self = new SelfClient({ apiKey: process.env.SELF_API_KEY! });
```

The SDK stores the key in memory and uses it on every call to `sessions.create(...)` and `sessions.get(...)`. See the [SDK reference](../sdk/nodejs.md).

## Rotation

Best practice: rotate live keys at least every 90 days, and immediately on any suspected exposure.

1. **Create key** → store the new value alongside the old one.
2. Roll the new value through your backends (typically a deploy with both `SELF_API_KEY_NEW` and `SELF_API_KEY_OLD` set, with traffic shifting).
3. Once 100% of traffic is on the new key, **Revoke** the old one in the dashboard.

Revoking is immediate. Within seconds, any request bearing the old key gets `401 unauthorized`.

## Revocation

You can revoke any key at any time from the list view. There's no undo. Revoked keys can't be re-issued; create a new one if needed.

## Audit

The keys list shows:

* **Created at**: when the key was issued.
* **Last used at**: last time we authenticated a request with it. `Never` for unused keys (a fingerprint of misconfigured deploys).
* **Created by**: which member issued it.

## Security notes

* Don't put keys in front-end code. They authorize session creation, which costs money.
* Don't commit keys. Use environment variables and secret managers.
* Per-service keys, not one shared key. If a service's key leaks, you only have to rotate that one.

## Related

* [SDK reference](../sdk/nodejs.md).
* [Test vs. live](../flows/test-vs-live.md): environments are baked into the key prefix.
