---
description: Sign and verify agent requests with the Self Agent ID SDKs
---

# SDK Integration

Self Agent ID provides three SDKs with identical feature parity:

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `@selfxyz/agent-sdk` | `npm install @selfxyz/agent-sdk` |
| Python | `selfxyz-agent-sdk` | `pip install selfxyz-agent-sdk` |
| Rust | `self-agent-sdk` | `cargo add self-agent-sdk` |

## Agent-Side: Signing Requests

Use `SelfAgent` to sign outbound API requests. The SDK attaches three headers to every request:

- `x-self-agent-address` — the agent's Ethereum address
- `x-self-agent-signature` — ECDSA signature of `keccak256(timestamp + METHOD + path + bodyHash)`
- `x-self-agent-timestamp` — Unix timestamp (seconds)

{% tabs %}
{% tab title="TypeScript" %}
```typescript
import { SelfAgent } from "@selfxyz/agent-sdk";

const agent = new SelfAgent({
  privateKey: process.env.AGENT_PRIVATE_KEY!,
  registryAddress: "0xaC3DF9ABf80d0F5c020C06B04Cced27763355944",
  rpcUrl: "https://forno.celo.org",
});

// Signed fetch — headers are attached automatically
const res = await agent.fetch("https://api.example.com/protected", {
  method: "POST",
  body: JSON.stringify({ action: "hello" }),
});
```
{% endtab %}

{% tab title="Python" %}
```python
from self_agent_sdk import SelfAgent

agent = SelfAgent(
    private_key=os.environ["AGENT_PRIVATE_KEY"],
    registry_address="0xaC3DF9ABf80d0F5c020C06B04Cced27763355944",
    rpc_url="https://forno.celo.org",
)

# Signed request
res = agent.fetch("https://api.example.com/protected", method="POST",
                   json={"action": "hello"})
```
{% endtab %}

{% tab title="Rust" %}
```rust
use self_agent_sdk::SelfAgent;

let agent = SelfAgent::new(
    &std::env::var("AGENT_PRIVATE_KEY")?,
    "0xaC3DF9ABf80d0F5c020C06B04Cced27763355944",
    "https://forno.celo.org",
)?;

let res = agent.fetch("https://api.example.com/protected")
    .method("POST")
    .json(&serde_json::json!({"action": "hello"}))
    .send()
    .await?;
```
{% endtab %}
{% endtabs %}

## Service-Side: Verifying Requests

Use `SelfAgentVerifier` to verify incoming agent requests. The verifier recovers the signer address from the ECDSA signature, derives the agent key, and checks `isVerifiedAgent()` on-chain.

### Builder Pattern

{% tabs %}
{% tab title="TypeScript" %}
```typescript
import { SelfAgentVerifier } from "@selfxyz/agent-sdk";

const verifier = SelfAgentVerifier.create()
  .requireAge(18)
  .requireOFAC()
  .maxAgentsPerHuman(1)
  .build();

// Express middleware
app.use("/api", verifier.auth());
```
{% endtab %}

{% tab title="Python" %}
```python
from self_agent_sdk import SelfAgentVerifier

verifier = (SelfAgentVerifier.create()
    .require_age(18)
    .require_ofac()
    .max_agents_per_human(1)
    .build())

# Flask middleware
@app.before_request
def verify_agent():
    result = verifier.verify_request(request)
    if not result.valid:
        return {"error": result.error}, 401
```
{% endtab %}

{% tab title="Rust" %}
```rust
use self_agent_sdk::SelfAgentVerifier;

let verifier = SelfAgentVerifier::builder()
    .require_age(18)
    .require_ofac()
    .max_agents_per_human(1)
    .build()?;

// Axum middleware (requires `axum` feature)
let app = Router::new()
    .route("/api/protected", post(handler))
    .layer(verifier.into_layer());
```
{% endtab %}
{% endtabs %}

### Direct Verification

```typescript
const result = await verifier.verify({
  signature: req.headers["x-self-agent-signature"],
  timestamp: req.headers["x-self-agent-timestamp"],
  method: req.method,
  url: req.url,
  body: req.body,
});

if (result.valid) {
  console.log(`Verified agent: ${result.agentAddress}, ID #${result.agentId}`);
  console.log(`Credentials:`, result.credentials);
}
```

## Security Defaults

`SelfAgentVerifier` defaults are strict:

| Setting | Default | Description |
|---------|---------|-------------|
| `requireSelfProvider` | `true` | Only accept proofs from Self Protocol's provider |
| `maxAgentsPerHuman` | `1` | One agent per human (sybil resistance) |
| Replay protection | Enabled | Signature nonce + timestamp freshness |
| Timestamp window | 300 seconds | Reject requests older than 5 minutes |

{% hint style="warning" %}
Setting `requireSelfProvider: false` accepts verification from any approved proof provider, not only Self Protocol.
{% endhint %}

## Agent Status & Info

```typescript
// Check registration status
const registered = await agent.isRegistered();

// Get full agent info
const info = await agent.getInfo();
// { agentId, address, agentKey, isVerified, proofProvider, proofExpiresAt }
```

## Proof Expiry

Agent proofs have a validity window. The SDK exposes expiry information and handles stale proofs:

```typescript
const info = await agent.getInfo();
const expiresAt = info.proofExpiresAt; // Unix timestamp (seconds)

// SDK verifiers automatically reject expired proofs
// 30-day warning threshold for upcoming expiry
```

{% hint style="warning" %}
When a proof expires, the agent's `isProofFresh()` returns `false` on-chain. There is no on-chain refresh mechanism — the agent must deregister and re-register with a fresh proof (scanning their passport again). The soulbound NFT is NOT burned — it remains as a historical record.
{% endhint %}

## A2A Agent Cards

Build and store standardized agent identity cards:

```typescript
import { buildAgentCard } from "@selfxyz/agent-sdk";

const card = await buildAgentCard(agent, {
  name: "My Agent",
  description: "A human-verified AI assistant",
});

// Store on-chain via updateAgentMetadata()
```

Cards follow the A2A format and are queryable via:
- `GET /api/cards/{chainId}/{agentId}`
- `GET /.well-known/a2a/{agentId}?chain={chainId}`
- `POST /api/a2a` — A2A JSON-RPC endpoint for agent-to-agent communication
