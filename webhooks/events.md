# Event catalog

Every event we send, with payload schemas.

## `verification.completed`

Fires when off-chain verification finishes, either successfully or with a definitive failure. This is the event most integrations care about.

### Payload

```json
{
  "type": "verification.completed",
  "verification_id": "ver_01HXYZ...",
  "external_uuid": "user_42",
  "flow_id": "9c0b4f1c-1d6c-4f1b-a8c4-9f0fa0a8d9e2",
  "flow_version_id": "fv_01H...",
  "environment": "live",
  "status": "valid",
  "proof_attributes": {
    "age_gte_18": true,
    "nationality_not_in_US": true
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
| `storage_state` | `'pending' \| 'committed' \| 'failed'` | Async storage state at time of fire. `pending` is common, wait for `verification.storage_committed` for the final state. |
| `storage_uri` | string \| null | Set when storage has committed. |

### Statuses

* `valid`: the proof verified and all flow predicates passed. This is the green-light case.
* `invalid`: the proof verified but at least one predicate failed (e.g. user is 16, flow requires 18).
* `error`: verification failed for a technical reason (unsupported document, malformed proof, signature mismatch). Not the user's fault necessarily.
* `expired`: session expired before completion.

### Event ID (for deduplication)

The deduplication key is `<verification_id>-<status>`.

---

## `verification.storage_committed`

Fires when the async decentralized-storage write for a verification succeeds.

### Payload

```json
{
  "type": "verification.storage_committed",
  "verification_id": "ver_01HXYZ...",
  "external_uuid": "user_42",
  "storage_uri": "ipfs://bafy...",
  "credential_id": "cred_01H...",
  "committed_at": "2026-05-29T17:33:25.110Z"
}
```

### When you'd care

* You want to reference the on-chain or decentralized-store credential ID for cross-chain attestation.
* You want a confirmation that the user's verified credential is durable, not just in our store.

If you don't care about decentralized storage, ignore this event type.

### Event ID (for deduplication)

`<verification_id>-storage-committed`.

---

## `verification.storage_failed`

Fires when the storage write permanently fails (after retries exhausted).

### Payload

```json
{
  "type": "verification.storage_failed",
  "verification_id": "ver_01HXYZ...",
  "external_uuid": "user_42",
  "error": "ipfs_pin_timeout",
  "failed_at": "2026-05-29T17:34:00.000Z"
}
```

### What it means

The verification itself is still authoritative, `verification.completed` already told you whether the user passed. Storage is a secondary durability layer; failure here doesn't invalidate the verification.

If your integration depends on the storage record (e.g. minting a credential NFT), you'll want to handle this case: log it, alert ops, or fall back to the API for the verification result.

### Event ID (for deduplication)

`<verification_id>-storage-failed`.

---

## Discriminated union

In TypeScript:

```ts
import type { WebhookEvent } from '@selfxyz/enterprise-sdk';

function handle(event: WebhookEvent) {
  switch (event.type) {
    case 'verification.completed':            // narrowed to VerificationCompletedPayload
    case 'verification.storage_committed':    // narrowed to VerificationStorageCommittedPayload
    case 'verification.storage_failed':       // narrowed to VerificationStorageFailedPayload
  }
}
```

## Forward compatibility

We may add fields to existing events without bumping major versions. The SDK's `webhookEvent` schema uses `.passthrough()` so unknown fields are preserved (and ignored by your existing code).

We may also add new event types over time. Subscribe explicitly to the ones you care about. That way your handler isn't surprised by a new type it doesn't know about.
