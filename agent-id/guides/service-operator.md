---
description: Protect your API by verifying agent requests against the on-chain registry
---

# Verifying Agents (Service Operator)

This guide shows how to add Self Agent ID verification to your API. After setup, only registered, human-backed agents can access your protected endpoints.

## 1. Install the SDK

```bash
npm install @selfxyz/agent-sdk    # TypeScript
pip install selfxyz-agent-sdk      # Python
cargo add self-agent-sdk           # Rust
```

## 2. Create a Verifier

Use the builder pattern to configure verification policy:

{% tabs %}
{% tab title="TypeScript" %}
```typescript
import { SelfAgentVerifier } from "@selfxyz/agent-sdk";

const verifier = SelfAgentVerifier.create()
  .network("mainnet")
  .requireAge(18)
  .requireOFAC()
  .requireSelfProvider()   // Ensure Self Protocol proofs (default: true)
  .sybilLimit(3)           // Max 3 agents per human
  .rateLimit({ perMinute: 10 })
  .build();
```
{% endtab %}

{% tab title="Python" %}
```python
from self_agent_sdk import SelfAgentVerifier

verifier = (SelfAgentVerifier.create()
    .network("mainnet")
    .require_age(18)
    .require_ofac()
    .require_self_provider()
    .sybil_limit(3)
    .rate_limit(per_minute=10)
    .build())
```
{% endtab %}

{% tab title="Rust" %}
```rust
use self_agent_sdk::SelfAgentVerifier;

let verifier = SelfAgentVerifier::create()
    .network("mainnet")
    .require_age(18)
    .require_ofac()
    .build();
```
{% endtab %}
{% endtabs %}

### Builder Options

| Method | Description | Default |
|--------|-------------|---------|
| `.network(name)` | `"mainnet"` or `"testnet"` | `"testnet"` |
| `.requireAge(n)` | Minimum age (18 or 21) | None |
| `.requireOFAC()` | OFAC sanctions screening | Off |
| `.requireNationality(...codes)` | Allowed ISO country codes | Any |
| `.requireSelfProvider()` | Require Self Protocol proofs | `true` |
| `.sybilLimit(n)` | Max agents per human (0 = unlimited) | `1` |
| `.rateLimit(config)` | Per-agent rate limiting | None |
| `.replayProtection(enabled?)` | Signature replay detection | `true` |
| `.includeCredentials()` | Attach credentials to request | `false` |
| `.maxAge(ms)` | Max signature age | `300000` (5 min) |
| `.cacheTtl(ms)` | On-chain query cache | `60000` (1 min) |

## 3. Add Middleware

{% tabs %}
{% tab title="Express (TypeScript)" %}
```typescript
import express from "express";

const app = express();
app.use(express.json());

// Protect all /api routes
app.use("/api", verifier.auth());

app.post("/api/data", (req, res) => {
  // req.agent is populated after verification
  console.log("Agent:", req.agent.address);
  console.log("Agent ID:", req.agent.agentId);
  console.log("Credentials:", req.agent.credentials);
  res.json({ ok: true });
});
```
{% endtab %}

{% tab title="Flask (Python)" %}
```python
from flask import Flask, g, jsonify
from self_agent_sdk.middleware.flask import require_agent

app = Flask(__name__)

@app.route("/api/data", methods=["POST"])
@require_agent(verifier)
def handle():
    print("Agent:", g.agent.agent_address)
    return jsonify(ok=True)
```
{% endtab %}

{% tab title="FastAPI (Python)" %}
```python
from fastapi import FastAPI, Depends
from self_agent_sdk.middleware.fastapi import AgentAuth

app = FastAPI()
agent_auth = AgentAuth(verifier)

@app.post("/api/data")
async def handle(agent=Depends(agent_auth)):
    print("Agent:", agent.agent_address)
    return {"ok": True}
```
{% endtab %}

{% tab title="Axum (Rust)" %}
```rust
use axum::{Router, routing::post, middleware, Json, Extension};
use self_agent_sdk::{VerifiedAgent, self_agent_auth};
use std::sync::Arc;
use tokio::sync::Mutex;

let verifier = Arc::new(Mutex::new(verifier));

let app = Router::new()
    .route("/api/data", post(handle))
    .layer(middleware::from_fn_with_state(verifier, self_agent_auth));

async fn handle(Extension(agent): Extension<VerifiedAgent>) -> Json<serde_json::Value> {
    println!("Agent: {:?}", agent.address);
    Json(serde_json::json!({ "ok": true }))
}
```
{% endtab %}
{% endtabs %}

## 4. Request Shape After Verification

After successful verification, `req.agent` (or equivalent) contains:

| Field | Type | Description |
|-------|------|-------------|
| `address` | `string` | Agent's Ethereum address |
| `agentId` | `number` | On-chain NFT token ID |
| `agentKey` | `string` | Derived `bytes32` key |
| `isVerified` | `boolean` | On-chain verification status |
| `proofProvider` | `string` | Provider contract address |
| `credentials` | `object` | ZK-attested credentials (if `includeCredentials()`) |

## 5. Credential-Based Access Control

Use credentials to gate by age, OFAC status, or nationality:

```typescript
const verifier = SelfAgentVerifier.create()
  .requireAge(21)                          // Must be 21+
  .requireOFAC()                           // Must pass OFAC screening
  .requireNationality("US", "GB", "DE")    // Only these countries
  .includeCredentials()                    // Attach credentials to request
  .build();

app.post("/api/restricted", verifier.auth(), (req, res) => {
  const { nationality, olderThan, ofac } = req.agent.credentials;
  // nationality: "US", olderThan: 21, ofac: [true, true, true]
});
```

## 6. Sybil Resistance

Control how many agents one human can use against your API:

| Setting | Behavior |
|---------|----------|
| `.sybilLimit(1)` | Strict — one agent per human (default) |
| `.sybilLimit(5)` | Moderate — up to 5 agents per human |
| `.sybilLimit(0)` | Detection only — unlimited, but `sameHuman()` available |

The verifier checks `getAgentCountForHuman(nullifier)` on-chain.

## 7. Provider Verification

{% hint style="danger" %}
**Security note**: Always use `requireSelfProvider()` (enabled by default) to ensure agent proofs came from Self Protocol. Without this check, a malicious provider could approve agents without real passport verification.
{% endhint %}

The verifier checks that `getProofProvider(agentId)` matches Self Protocol's provider address.

## 8. Replay Protection + Rate Limiting

**Replay protection** (enabled by default):
- Caches `{signature + timestamp}` hashes (10,000 entries)
- Same signature cannot be used twice

**Rate limiting** (optional):
```typescript
.rateLimit({ perMinute: 10, perHour: 100 })
```
Per-agent sliding-window rate limits.

## 9. Error Handling

The middleware returns standard HTTP errors:

| Status | Meaning |
|--------|---------|
| `401` | Missing or invalid signature headers |
| `403` | Agent not verified, failed policy check, or rate limited |
| `500` | On-chain query failed |

Handle these in your client:

```typescript
const res = await agent.fetch("https://api.example.com/data", { method: "POST" });
if (res.status === 403) {
  const error = await res.json();
  // { error: "Agent not verified on-chain" }
  // { error: "Age requirement not met: requires 18, has 0" }
  // { error: "Rate limit exceeded" }
}
```

## Next Steps

- [Build an agent that calls your API](agent-builder.md)
- [Gate smart contracts by agent ID](contract-developer.md)
- [Troubleshooting](../troubleshooting.md)
