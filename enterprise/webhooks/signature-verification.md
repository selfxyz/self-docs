# Signature verification

Every webhook delivery is signed using [Svix](https://www.svix.com)'s signing scheme. Verify the signature before trusting the payload.

## What to verify

* The body matches the signature.
* The timestamp is recent (defends against replay).

Svix's libraries do both for you. We strongly recommend using a library; rolling your own HMAC check is a footgun.

## Node (recommended path)

The SDK wraps Svix:

```ts
import { SelfWebhooks } from '@selfxyz/enterprise-sdk';

const event = SelfWebhooks.verify(rawBody, headers, secret);
```

See [SDK: Verify webhooks](../sdk/verify-webhooks.md) for the full setup including raw-body wiring.

If you'd rather use Svix directly:

```ts
import { Webhook } from 'svix';

const wh = new Webhook(process.env.SELF_WEBHOOK_SECRET!);
const event = wh.verify(rawBody, headers);   // throws on bad signature
```

## Python

```python
from svix import Webhook, WebhookVerificationError

wh = Webhook(os.environ["SELF_WEBHOOK_SECRET"])
try:
    event = wh.verify(raw_body, headers)
except WebhookVerificationError:
    return Response(status=400)
```

## Ruby

```ruby
require 'svix'

wh = Svix::Webhook.new(ENV['SELF_WEBHOOK_SECRET'])
event = wh.verify(raw_body, headers)  # raises Svix::WebhookVerificationError
```

## Go

```go
import "github.com/svix/svix-webhooks/go"

wh, err := svix.NewWebhook(os.Getenv("SELF_WEBHOOK_SECRET"))
if err != nil { /* ... */ }

err = wh.Verify(rawBody, headers)
if err != nil {
    http.Error(w, "bad signature", http.StatusBadRequest)
    return
}
```

## Rust

```rust
use svix::webhooks::Webhook;

let wh = Webhook::new(&env::var("SELF_WEBHOOK_SECRET")?)?;
wh.verify(raw_body, &headers)?;
```

## Manual verification

If you absolutely cannot use a Svix library, the signing scheme is documented at [docs.svix.com/receiving/verifying-payloads/how-manual](https://docs.svix.com/receiving/verifying-payloads/how-manual).

Summary:

```
signed_payload = svix-id + "." + svix-timestamp + "." + body
signature      = base64(HMAC-SHA256(secret_bytes, signed_payload))
```

The header may contain multiple comma-separated signatures (each prefixed with a version, e.g. `v1,<base64>`); your payload matches if any one of them matches.

You **must** also enforce a timestamp tolerance (Svix's default is 5 minutes) to prevent replay attacks.

## Rotating the secret

You can rotate a webhook secret in **Settings → Webhooks → \[endpoint\] → Rotate secret**. For 24 hours both old and new secrets verify; after that, only the new one. See [Dashboard: Webhooks → Rotation](../dashboard/webhooks.md#rotation).

To handle the overlap window without code changes, the Svix Node library can be initialized with multiple secrets, but the SDK doesn't expose this directly. The simplest pattern: roll the new secret in via env, then redeploy.

## Common failure modes

* **Parsed body.** Verification runs on bytes; if your framework JSON-parsed the body, the canonical form is lost. Capture `rawBody` before parsing.
* **Trimming or transforming.** A trailing newline added by your proxy will break the signature. Configure the proxy to pass the body unchanged.
* **Wrong secret.** Each endpoint has its own. Triple-check you're using the right one.
* **Clock skew.** A drifted server clock can fail the timestamp tolerance check. Make sure NTP is healthy.
