---
description: The ERC-8004 registration document and A2A Agent Card served at an agent's agentURI
---

# Agent Registration JSON & Agent Cards

Every agent registered with `SelfAgentRegistry` can publish a JSON document at its `agentURI`. This file is the agent's identity document: it is how services, indexers, and [8004scan](https://8004scan.xyz) discover and verify the agent's capabilities.

## How it relates to A2A

The document at `agentURI` is an ERC-8004 registration file. When the A2A fields (`version`, `url`, `provider`, `capabilities`, `securitySchemes`) are included, the **same** document is simultaneously a valid A2A Agent Card. There is no second document and no separate A2A registration step. The two specs read non-overlapping fields and ignore what they do not recognize.

```
ERC-8004 reads: type, name, description, image, services[], active, registrations[], supportedTrust[]
A2A reads:      name, description, url, version, provider, capabilities, securitySchemes,
                skills[], defaultInputModes, defaultOutputModes
```

## Minimal ERC-8004 format

Enough for the on-chain registry and 8004scan:

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "My Agent",
  "description": "What this agent does",
  "image": "https://example.com/avatar.png",
  "services": [
    {
      "name": "A2A",
      "endpoint": "https://my-agent.example.com/a2a",
      "version": "1.0"
    }
  ]
}
```

## Combined ERC-8004 + A2A format

Adding the A2A fields makes this single document a valid A2A Agent Card too:

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "My Agent",
  "description": "A human-backed AI agent verified via Self Protocol",
  "image": "https://my-agent.example.com/avatar.png",
  "version": "0.1.0",
  "url": "https://my-agent.example.com/a2a",
  "provider": { "name": "Acme Corp", "url": "https://acme.example.com" },
  "capabilities": { "streaming": false, "pushNotifications": false },
  "securitySchemes": [
    { "type": "bearer", "description": "API key in Authorization header" }
  ],
  "active": true,
  "services": [
    { "name": "A2A", "endpoint": "https://my-agent.example.com/a2a", "version": "1.0" },
    { "name": "MCP", "endpoint": "https://my-agent.example.com/mcp", "version": "1.0" }
  ],
  "registrations": [
    {
      "agentId": 42,
      "agentRegistry": "eip155:42220:0xaC3DF9ABf80d0F5c020C06B04Cced27763355944"
    }
  ],
  "supportedTrust": ["reputation"]
}
```

## Field reference

| Field | Required | Protocol | Notes |
|-------|----------|----------|-------|
| `type` | YES | ERC-8004 | Must be `"https://eips.ethereum.org/EIPS/eip-8004#registration-v1"` |
| `name` | YES | ERC-8004 + A2A | Human-readable agent name |
| `description` | YES | ERC-8004 + A2A | What the agent does |
| `image` | YES | ERC-8004 | Avatar URL |
| `services` | YES | ERC-8004 | At least one service endpoint |
| `services[].name` | YES | ERC-8004 | One of: `web`, `A2A`, `MCP`, `OASF`, `ENS`, `DID`, `email` |
| `services[].endpoint` | YES | ERC-8004 | Service URI |
| `services[].version` | YES (for A2A/MCP) | ERC-8004 | Required by A2A protocol, e.g. `"1.0"` |
| `active` | NO | ERC-8004 | Set `false` when the agent is inactive (e.g. proof expired) |
| `registrations` | NO | ERC-8004 | Cross-chain registry refs using CAIP-10: `eip155:<chainId>:<address>` |
| `supportedTrust` | NO | ERC-8004 | `reputation`, `crypto-economic`, `tee-attestation` |
| `version` | NO\* | A2A | Agent **software** version (e.g. `"0.1.0"`). \*Required for an A2A Agent Card |
| `url` | NO\* | A2A | A2A primary endpoint. MUST equal `services[name="A2A"].endpoint`. \*Required for an A2A Agent Card |
| `provider` | NO\* | A2A | Publisher identity `{ name, url?, email? }`. \*Required for an A2A Agent Card |
| `capabilities` | NO\* | A2A | `{ streaming: bool, pushNotifications: bool }`. \*Required for an A2A Agent Card |
| `securitySchemes` | NO\* | A2A | Auth methods. `{ type: "bearer" \| "apiKey" \| "oauth2" \| "none" }[]`. \*Required for an A2A Agent Card |
| `defaultInputModes` | NO | A2A | MIME types the agent accepts, e.g. `["text/plain"]` |
| `defaultOutputModes` | NO | A2A | MIME types the agent produces |
| `skills` | NO | A2A | `[{ name, description? }]` — specific tasks the agent offers |

## Constraints when combining the two

- `url` (the A2A primary endpoint) and `services[name="A2A"].endpoint` **MUST** point to the same address. A2A clients read `url`; ERC-8004 indexers read `services`.
- `version` = agent **software** version (e.g. `"0.1.0"`); `services[].version` = **protocol** version (e.g. `"1.0"`). These are distinct fields.
- The path `/.well-known/agent-card.json` may serve this same document for A2A well-known discovery — no duplication required.

## Agents minted via Self Protocol start with a blank agentURI

{% hint style="warning" %}
Agents registered through the **Hub V2 Self Protocol flow** (the `verifySelfProof` callback that mints the NFT) are created with a **blank `agentURI`**. The on-chain `Registered` event carries an empty string for the URI.
{% endhint %}

You **must** call `setAgentURI()` after registration:

```solidity
registry.setAgentURI(agentId, "https://my-agent.example.com/.well-known/agent-card.json");
```

Until `setAgentURI()` is called:

- 8004scan indexes the agent with no identity document
- A2A clients cannot discover your service endpoints
- The `active` and `services` fields are invisible to the ecosystem

This does not apply to agents registered via the `register(agentURI)` or `registerWithHumanProof(agentURI, ...)` overloads, which accept a URI directly.

## On-chain metadata

When registered via `SelfAgentRegistry`, agents can carry key-value metadata set with `setMetadata(agentId, key, value)`:

| Key | Description |
|-----|-------------|
| `agentWallet` | Reserved. Set via `setAgentWallet()` — a payment address separate from the NFT owner, with an EIP-712 signature from the new wallet. |

Any custom key is allowed except the reserved `agentWallet`.

## SDK helper

The TypeScript SDK builds valid documents with `generateRegistrationJSON`:

```typescript
import { generateRegistrationJSON } from "@selfxyz/agent-sdk";

// Minimal ERC-8004 only
const minimalDoc = generateRegistrationJSON({
  name: "My Agent",
  description: "What it does",
  image: "https://...",
  services: [{ name: "A2A", endpoint: "https://...", version: "1.0" }],
});

// Combined ERC-8004 + A2A (single document, both protocols)
const fullDoc = generateRegistrationJSON({
  name: "My Agent",
  description: "What it does",
  image: "https://...",
  services: [
    { name: "A2A", endpoint: "https://my-agent.example.com/a2a", version: "1.0" },
  ],
  a2a: {
    version: "0.1.0",
    url: "https://my-agent.example.com/a2a",
    provider: { name: "Acme Corp" },
    capabilities: { streaming: false, pushNotifications: false },
    securitySchemes: [{ type: "bearer" }],
  },
});
```

## Hosting requirements

The JSON file must be:

1. Accessible over HTTPS at the `agentURI` provided during registration
2. Served with `Content-Type: application/json`
3. Updated whenever the agent's services or status change (update the document, then call `setAgentURI()` if the URL itself moved)

## Validation

Use the 8004scan validator at [8004scan.xyz](https://8004scan.xyz) to confirm your registration JSON is correctly formatted before registering.

## Where the API serves cards from

The API reads the on-chain `agentMetadata` and serves the stored card as HTTP JSON, so you do not have to host anything yourself if you set the card on-chain:

- `GET /api/cards/{chainId}/{agentId}` — the Agent Card JSON
- `GET /.well-known/a2a/{agentId}?chain={chainId}` — redirects to the card resolver

See [REST API](rest-api.md) and [SDK Integration](sdk-integration.md#a2a-agent-cards).
