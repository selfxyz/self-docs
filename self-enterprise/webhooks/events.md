# Event catalog

Self delivers a single webhook event: `verification.completed`. Every registered endpoint receives it; there's no per-event subscription.

## `verification.completed`

Fires when off-chain verification finishes, either successfully or with a definitive failure. This is the event integrations care about.

### Payload

```json
{
  "type": "verification.completed",
  "verification_id": "7f3b2a1e-9c4d-4b2a-8e1f-2c6d5a4b3c2d",
  "external_uuid": "a1b2c3d4-5678-4e9a-b012-3456789abcde",
  "flow_id": "9c0b4f1c-1d6c-4f1b-a8c4-9f0fa0a8d9e2",
  "flow_version_id": "b1e2c3d4-5678-4abc-9def-0123456789ab",
  "environment": "live",
  "status": "valid",
  "proof_attributes": {
    "age_gte_18": true,
    "country_allowed": true,
    "ofac_clear": true
  },
  "proof": { /* raw ZK proof JSON, or null if status != 'valid' */ },
  "verified_at": "2026-05-29T17:33:21.412Z",
  "storage_state": "pending",
  "storage_uri": null
}
```

### Fields

| Field | Type | Notes |
| --- | --- | --- |
| `verification_id` | string | Stable, unique. Use as your dedup key. |
| `external_uuid` | string | The identifier you passed when creating the session. |
| `flow_id` / `flow_version_id` | string | Which flow + frozen version this verification ran against. |
| `environment` | `'test' \| 'live'` | Matches the API key environment that created the session. |
| `status` | `'valid' \| 'invalid' \| 'error' \| 'expired'` | See [statuses](#statuses) below. |
| `proof_attributes` | object | The disclosed predicates (booleans for predicates, values for reveals). Empty object on non-`valid` statuses. |
| `proof` | object \| null | Raw Groth16 proof JSON. Present when `status === 'valid'`. Most integrations ignore this; we already verified it. |
| `verified_at` | ISO-8601 | When verification completed. |
| `storage_state` | `'pending' \| 'committed' \| 'failed'` | Async storage state at the time the event fired, usually `pending`. For the final state, read the session with [`sessions.get(...)`](../sdk/nodejs.md). |
| `storage_uri` | string \| null | Set once storage has committed, usually `null` at fire time. |

### Statuses

* `valid`: the proof verified and all flow predicates passed. This is the green-light case.
* `invalid`: the proof verified but at least one predicate failed (e.g. user is 16, flow requires 18).
* `error`: verification failed for a technical reason (unsupported document, malformed proof, signature mismatch). Not the user's fault necessarily.
* `expired`: session expired before completion.

### Event ID (for deduplication)

The deduplication key is `<verification_id>-completed` (the same for every status, so a retry of this event reuses it).

## Handling in TypeScript

The payload is typed as a discriminated union on `event.type`, so branch on it. Today `verification.completed` is the only type delivered:

```ts
import type { WebhookEvent } from '@selfxyz/enterprise-sdk';

function handle(event: WebhookEvent) {
  if (event.type === 'verification.completed') {
    // narrowed to VerificationCompletedPayload
    // event.verification_id, event.external_uuid, event.proof_attributes, event.status
  }
}
```

## Forward compatibility

We may add fields to existing events without bumping major versions. The SDK's `webhookEvent` schema uses `.passthrough()`, so unknown fields are preserved (and ignored by your existing code). We may also add new event types over time, so branch on `event.type` and let your handler ignore any type it doesn't recognize.
