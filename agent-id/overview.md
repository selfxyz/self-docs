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

## Networks

| Network | Chain ID | Registry Address |
|---------|----------|-----------------|
| Celo Mainnet | `42220` | `0x62e37d0f6c5f67784b8828b3df68bcdbb2e55095` |
| Celo Sepolia | `11142220` | `0x42cea1b318557ade212bed74fc3c7f06ec52bd5b` |

{% hint style="warning" %}
Celo Sepolia chain ID is **11142220**, not 44787 (deprecated Alfajores).
{% endhint %}

## Live Deployment

- Web app: [https://selfagentid.xyz](https://selfagentid.xyz)
- GitHub: [https://github.com/selfxyz/self-agent-id](https://github.com/selfxyz/self-agent-id)

## Next Steps

- [Registration Modes](registration-modes.md) — Choose how to register your agent
- [SDK Integration](sdk-integration.md) — Sign and verify agent requests
- [Smart Contracts](smart-contracts.md) — On-chain gating and queries
