---
description: End-to-end guide for building AI agents with proof-of-human identity
---

# Building an Agent

This guide walks through registering an AI agent with Self Agent ID and making authenticated requests. By the end, your agent will have an on-chain identity backed by a real passport verification.

## 1. Choose Your SDK

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `@selfxyz/agent-sdk` | `npm install @selfxyz/agent-sdk` |
| Python | `selfxyz-agent-sdk` | `pip install selfxyz-agent-sdk` |
| Rust | `self-agent-sdk` | `cargo add self-agent-sdk` |

All three SDKs have identical functionality with language-idiomatic naming.

## 2. Register Your Agent

Six registration modes — choose based on your use case:

| Mode | Best for | Wallet needed? |
|------|----------|:-:|
| `wallet-free` | Embedded agents, IoT, CLI-only | No |
| `ed25519` | OpenClaw, Eliza, IronClaw agents | No |
| `linked` | Autonomous AI agents | Yes (human's) |
| `ed25519-linked` | Ed25519 agents with human wallet | Yes (human's) |
| `privy` | Social login (Google, Twitter) | No |
| `smartwallet` | Consumer-facing, passkey UX | No |

### Via the SDK (simplest)

Render the passport-scan QR in your own frontend and read the result from the chain. No hosted service is involved. This is the recommended path. See [Register Without the Web App](../register-without-the-app.md) for the full runnable example (generate the agent key, sign the challenge, build the QR with `@selfxyz/qrcode`, mint on-chain).

### Via CLI

```bash
# Install the CLI (comes with the SDK)
npm install -g @selfxyz/agent-sdk

# Initialize registration
self-agent register init \
  --mode linked \
  --human-address 0xYourWallet \
  --network mainnet \
  --minimum-age 18 \
  --ofac

# Print the QR (fetched from the API) to scan with the Self app
self-agent register open --session .self/session-*.json

# Wait for verification to complete
self-agent register wait --session .self/session-*.json

# Export the agent private key
self-agent register export --session .self/session-*.json --unsafe --print-private-key
```

The CLI talks to `https://agent-api.self.xyz` by default (override with `SELF_AGENT_API_BASE`).

### Via A2A Protocol (for agents)

Agents can self-register by sending a JSON-RPC request to the A2A endpoint:

```bash
curl -X POST https://agent-api.self.xyz/api/a2a \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "parts": [{ "type": "data", "data": { "intent": "register" } }]
      }
    }
  }'
```

The endpoint returns a QR code and deep link. A human scans the QR with the Self app to complete verification. Send `{ "intent": "help" }` to see all available modes and a decision guide.

### Via REST API

```bash
curl -X POST https://agent-api.self.xyz/api/agent/register \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "linked",
    "network": "mainnet",
    "humanAddress": "0xYourWallet",
    "disclosures": { "minimumAge": 18, "ofac": true }
  }'
```

Poll `/api/agent/register/status?token=<token>` until `stage: "completed"`.

### Via Smart Wallet (passkeys)

Smart-wallet mode uses a passkey to create a ZeroDev Kernel smart account as the guardian, with gasless operations via the Pimlico paymaster on mainnet. Build the passkey step into your own frontend with `@zerodev/sdk` and `@zerodev/passkey-validator`; the agent keypair and challenge are generated the same way as `linked`. The gasless bundler and paymaster are proxied through `https://agent-api.self.xyz/api/aa/*`. See [Registration Modes](../registration-modes.md#smart-wallet).

## 3. Sign Outbound Requests

Every SDK provides `agent.fetch()` which automatically signs requests with three headers:

{% tabs %}
{% tab title="TypeScript" %}
```typescript
import { SelfAgent } from "@selfxyz/agent-sdk";

const agent = new SelfAgent({
  privateKey: process.env.AGENT_PRIVATE_KEY!,
  network: "mainnet",
});

const res = await agent.fetch("https://api.example.com/data", {
  method: "POST",
  body: JSON.stringify({ query: "test" }),
});
```
{% endtab %}

{% tab title="Python" %}
```python
from self_agent_sdk import SelfAgent
import os

agent = SelfAgent(
    private_key=os.environ["AGENT_PRIVATE_KEY"],
    network="mainnet",
)

res = agent.fetch("https://api.example.com/data",
                   method="POST", body='{"query": "test"}')
```
{% endtab %}

{% tab title="Rust" %}
```rust
use self_agent_sdk::{SelfAgent, SelfAgentConfig, NetworkName};

let agent = SelfAgent::new(SelfAgentConfig {
    private_key: std::env::var("AGENT_PRIVATE_KEY").unwrap(),
    network: Some(NetworkName::Mainnet),
    registry_address: None,
    rpc_url: None,
}).unwrap();

let res = agent.fetch(
    "https://api.example.com/data",
    Some(reqwest::Method::POST),
    Some(r#"{"query":"test"}"#.to_string()),
).await.unwrap();
```
{% endtab %}
{% endtabs %}

The signed headers:

| Header | Value |
|--------|-------|
| `x-self-agent-address` | Agent's Ethereum address |
| `x-self-agent-signature` | ECDSA signature of `keccak256(timestamp + METHOD + path + bodyHash)` |
| `x-self-agent-timestamp` | Unix timestamp (ms) |

## 4. Check Registration Status

```typescript
const registered = await agent.isRegistered();
const info = await agent.getInfo();
// { agentId, isVerified, proofProvider, verificationStrength, ... }
```

## 5. Read Credentials

```typescript
const creds = await agent.getCredentials();
// { nationality, olderThan, ofac, dateOfBirth, gender, issuingState, ... }
```

Credentials are ZK-attested — extracted from passport data without revealing the full document.

## 6. Set an Agent Card (A2A)

Agent cards enable discovery in agent-to-agent protocols:

```typescript
await agent.setAgentCard({
  name: "My Agent",
  description: "Analyzes market data",
  url: "https://myagent.example.com",
  skills: [{ name: "market-analysis", description: "Analyzes crypto markets" }],
});

const card = await agent.getAgentCard();
```

Cards are stored on-chain and readable by any agent or service.

## Gotchas

{% hint style="warning" %}
**`userDefinedData` is UTF-8, not raw bytes.** The Self SDK passes `userDefinedData` as a UTF-8 string. Each byte position uses the ASCII character (`'0'` not `0x00`). This is the #1 integration mistake.
{% endhint %}

{% hint style="warning" %}
**Mainnet uses real passports. Testnet uses mock documents.** Registration defaults to Celo Mainnet (chain ID 42220) which requires a real passport scan via the Self app. Use `network: "testnet"` (chain ID 11142220) for testing — testnet also requires the Self app, but you can generate mock documents within the app instead of using a real passport.
{% endhint %}

{% hint style="info" %}
**Key management.** Store your agent private key securely. The CLI exports keys with `--unsafe` for a reason. Use environment variables or a secrets manager in production.
{% endhint %}

## Next Steps

- [Verify agent requests in your API](service-operator.md)
- [Gate smart contracts by agent ID](contract-developer.md)
- [SDK README (TypeScript)](https://github.com/selfxyz/self-agent-id/tree/main/typescript-sdk)
- [SDK README (Python)](https://github.com/selfxyz/self-agent-id/tree/main/python-sdk)
- [SDK README (Rust)](https://github.com/selfxyz/self-agent-id/tree/main/rust-sdk)
