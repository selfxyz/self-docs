---
description: On-chain proof-of-human identity for AI agents
---

# Overview

Self Agent ID is an on-chain identity registry that binds AI agent identities to Self Protocol human proofs. Each agent receives a **soulbound ERC-721 NFT** backed by a ZK passport verification, enabling trustless proof-of-human for autonomous agents.

The system implements the [ERC-8004 Proof-of-Human extension](https://eips.ethereum.org/EIPS/eip-8004) and provides SDK implementations in TypeScript, Python, and Rust with identical feature parity.

## Who This Is For

| Audience | What You Do |
|----------|-------------|
| **Agent builders** | Register an agent identity, sign outbound requests with `SelfAgent` |
| **Service/API teams** | Verify inbound agent signatures with `SelfAgentVerifier` middleware |
| **Protocol/infra teams** | Gate smart contracts, query on-chain state, compose with registry interfaces |

## How It Works

1. **Human scans passport** with the Self app (ZK proof generated locally on phone)
2. **Hub V2 verifies the proof** on-chain and calls back to the registry
3. **Registry mints a soulbound NFT** binding the agent key to a unique human nullifier
4. **Agent signs requests** using the SDK; services verify signatures against the on-chain registry

{% hint style="info" %}
No personal data ever leaves the user's device. Only a ZK proof and nullifier are submitted on-chain.
{% endhint %}

## Key Properties

- **Trustless**: On-chain verification with no central authority
- **Private**: ZK proofs reveal nothing about the human's identity
- **Composable**: A single registry call integrates into any EVM contract, backend, or agent framework
- **Sybil-resistant**: Each human maps to a unique nullifier, preventing unlimited agent registration
- **Time-bounded**: Proofs expire (default: 1 year or document expiry), requiring periodic re-authentication
- **ERC-8004 compliant**: Implements the standard Identity Registry interface with a Proof-of-Human extension

## Networks

| Network | Chain ID | Registry Address | Passport Required |
|---------|----------|-----------------|-------------------|
| Celo Mainnet (default) | `42220` | `0xaC3DF9ABf80d0F5c020C06B04Cced27763355944` | Real passport via Self app |
| Celo Sepolia (testnet) | `11142220` | `0x043DaCac8b0771DD5b444bCC88f2f8BBDBEdd379` | Mock documents via Self app |

{% hint style="warning" %}
**Mainnet is the default.** Registration requires a real passport scanned with the Self app. Testnet also requires the Self app, but you can generate mock documents within the app instead of using a real passport. Testnet is for development and testing only.
{% endhint %}

{% hint style="warning" %}
Celo Sepolia chain ID is **11142220**, not 44787 (deprecated Alfajores).
{% endhint %}

## Live Deployment

- Web app: [https://selfagentid.xyz](https://selfagentid.xyz)
- GitHub: [https://github.com/selfxyz/self-agent-id](https://github.com/selfxyz/self-agent-id)

## A2A Protocol

Agents can interact programmatically via the A2A JSON-RPC endpoint at `/api/a2a`. This supports registration, verification, lookup, deregistration, and proof freshness checks — all through structured intents or natural language.

Send `{ "intent": "help" }` to get a full list of capabilities and a decision guide for choosing the right registration mode.

## Architecture

The system consists of three on-chain registries:

- **SelfAgentRegistry** — Core identity registry (ERC-8004 + Proof-of-Human extension, soulbound ERC-721)
- **SelfReputationRegistry** — ERC-8004 Reputation Registry for aggregated feedback with document-type weighted signals
- **SelfValidationRegistry** — ERC-8004 Validation Registry for on-demand third-party validation

Plus SDKs in TypeScript, Python, and Rust with identical feature parity for signing requests, verifying agents, and interacting with the registry.

## Next Steps

- [Registration Modes](registration-modes.md) — Choose how to register your agent
- [SDK Integration](sdk-integration.md) — Sign and verify agent requests
- [Smart Contracts](smart-contracts.md) — On-chain gating, queries, and the full ERC-8004 interface
- [A2A Protocol & REST API](rest-api.md) — Programmatic agent interaction via JSON-RPC
