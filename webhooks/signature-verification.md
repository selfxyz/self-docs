# Signature verification

Every webhook delivery is signed using HMAC-SHA256. Verify the signature before trusting the payload.

## What to verify

* The body matches the signature.
* The timestamp is recent (defends against replay).

The Enterprise SDK does both for you. We strongly recommend using it. Rolling your own HMAC check is a footgun.

## Node (recommended path)

The SDK does it in one call:

```ts
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

const event = SelfWebhooks.verify(rawBody, headers, secret);
```

See [SDK: Verify webhooks](../sdk/verify-webhooks.md) for the full setup including raw-body wiring.

## Other languages

Official SDKs for Python, Go, Ruby, and Rust are on the roadmap. In the meantime, verify manually using HMAC-SHA256 (see below) or contact support@self.xyz for a verification helper.

## Manual verification

The signing scheme:

```
signed_payload = <svix-id> + "." + <svix-timestamp> + "." + <body>
signature      = base64(HMAC-SHA256(secret_bytes, signed_payload))
```

Where `<svix-id>`, `<svix-timestamp>`, and `<body>` come from the request:

* `<svix-id>` — the `svix-id` HTTP header.
* `<svix-timestamp>` — the `svix-timestamp` HTTP header (Unix seconds).
* `<body>` — the raw request body (UTF-8 bytes, exactly as received).

The `svix-signature` header may contain multiple comma-separated signatures (each prefixed with a version, e.g. `v1,<base64>`). Your computed signature matches if any one of them matches.

You **must** also enforce a timestamp tolerance (default: 5 minutes) to prevent replay attacks. Compare `svix-timestamp` against the current time and reject deliveries outside the window.

The webhook signing secret (`whsec_...`) is the raw secret bytes after stripping the `whsec_` prefix and base64-decoding the remainder.

## Rotating the secret

You can rotate a webhook secret in **Settings → Webhooks → \[endpoint\] → Rotate secret**. For 24 hours both old and new secrets verify; after that, only the new one. See [Dashboard: Webhooks → Rotation](../dashboard/webhooks.md#rotation).

To handle the overlap window without code changes, roll the new secret in via env and redeploy within the 24-hour window.

## Common failure modes

* **Parsed body.** Verification runs on bytes; if your framework JSON-parsed the body, the canonical form is lost. Capture `rawBody` before parsing.
* **Trimming or transforming.** A trailing newline added by your proxy will break the signature. Configure the proxy to pass the body unchanged.
* **Wrong secret.** Each endpoint has its own. Triple-check you're using the right one.
* **Clock skew.** A drifted server clock can fail the timestamp tolerance check. Make sure NTP is healthy.
