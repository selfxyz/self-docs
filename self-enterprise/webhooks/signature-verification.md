# Signature verification

Every webhook delivery is signed with HMAC-SHA256. Verify the signature before trusting the payload, and don't roll your own check, the SDK does it correctly in one call.

## What the SDK verifies

* The body matches the signature.
* The timestamp is recent (a 5-minute tolerance, which defends against replay).

## Node

```ts
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

const event = SelfWebhooks.verify(rawBody, headers, secret);
```

`SelfWebhooks.verify` reads the signature off the request headers for you, so just pass the request headers through. See [SDK: Verify webhooks](../sdk/verify-webhooks.md) for the full setup including raw-body wiring.

## Rotating the secret

If a signing secret is rotated, the new `whsec_...` is revealed once, the same way it is when the endpoint is first created. Roll it into your handler by updating `SELF_WEBHOOK_SECRET` and redeploying.

## Common failure modes

* **Parsed body.** Verification runs on bytes; if your framework JSON-parsed the body, the canonical form is lost. Capture `rawBody` before parsing.
* **Trimming or transforming.** A trailing newline added by your proxy will break the signature. Configure the proxy to pass the body unchanged.
* **Wrong secret.** Each endpoint has its own. Triple-check you're using the right one.
* **Clock skew.** A drifted server clock can fail the timestamp tolerance check. Make sure NTP is healthy.
