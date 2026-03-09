---
description: Six registration modes for AI agents with proof-of-human verification
---

# Registration Modes

Self Agent ID supports six registration modes. All produce the same on-chain result — a soulbound NFT with ZK proof-of-human — but they differ in key type, wallet requirements, and user experience. Two modes (privy and smartwallet) are UI wrappers that use the same underlying contract flows.

## Networks

| Network | Chain ID | Default? | Passport Type | Use Case |
|---------|----------|----------|---------------|----------|
| **Celo Mainnet** | 42220 | Yes | Real passports via Self app | Production deployments |
| **Celo Sepolia** | 11142220 | No | Mock documents via Self app | Testing and development |

{% hint style="warning" %}
Mainnet is the default network. Real passport verification requires the Self mobile app and a physical passport with NFC. Testnet also requires the Self app, but you can generate mock documents within the app instead of using a real passport. Testnet should only be used for development and testing.
{% endhint %}

## Quick Decision Guide

```
                  ┌─────────────────────┐
                  │ What key type does   │
                  │ your agent use?      │
                  └────────┬────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         Ed25519                     ECDSA
              │                         │
     ┌────────┴────────┐      ┌────────┴─────────────┐
     │ Need human      │      │ How should the       │
     │ wallet binding? │      │ human authenticate?  │
     └───┬─────────┬───┘      └──┬──────┬──────┬─────┘
         │         │             │      │      │
        No        Yes         Wallet  Social  Passkey
         │         │          required login
         ▼         ▼             │      │      │
    ed25519   ed25519-        linked   ▼      ▼
              linked      (Recommended) privy  smartwallet
                                   │
                              No wallet at all?
                                   │
                                   ▼
                              wallet-free
```

## Mode Comparison

| Mode | Wallet Required? | NFT Owner | Agent Key Type | Best For |
|------|-----------------|-----------|----------------|----------|
| **linked** (Recommended) | Yes (registration only) | Human's wallet | Independent ECDSA keypair | Autonomous agents with human oversight |
| **wallet-free** | No | Agent's address | Independent ECDSA keypair | Non-crypto users, embedded agents |
| **ed25519** | No | Derived from pubkey | Ed25519 keypair | OpenClaw/Eliza/IronClaw agents |
| **ed25519-linked** | Yes (registration only) | Human's wallet | Ed25519 keypair | Ed25519 agents with human wallet binding |
| **privy** | No (social login) | Human's wallet | Independent ECDSA keypair | OAuth users (Google, Twitter, etc.) |
| **smartwallet** | No (passkey) | Agent's address | Independent ECDSA keypair | Passkey UX, gasless management |

## Linked (Recommended)

The agent generates its own dedicated ECDSA keypair. The human's wallet is used only during registration to establish ownership.

**Flow:**

1. Connect browser wallet
2. A fresh agent keypair is generated in the browser
3. Agent signs a challenge proving key ownership (ECDSA)
4. Scan passport with Self app — contract verifies both ZK proof and agent signature in one step
5. Human's wallet key is never exposed to agent software

**Use when:** You're building autonomous agents that need their own identity, or you want key rotation without re-registering the human.

{% hint style="info" %}
Linked mode is the recommended default for most agent deployments. It separates the human's wallet from the agent's operational key while maintaining a verifiable ownership link.
{% endhint %}

## Wallet-Free

No crypto wallet required. The agent generates an ECDSA keypair and owns its own NFT.

**Flow:**

1. Agent keypair generated in browser
2. Agent signs a challenge proving key ownership
3. Scan passport with Self app — NFT is minted to the agent's own address
4. Deregister anytime by scanning passport again (ZK proof links to your unique identity)

**Use when:** The user has no crypto wallet and just needs a quick registration with their passport, or you're embedding agent identity into a service where wallet management is undesirable.

{% hint style="info" %}
In wallet-free mode, the user must save the agent's private key securely. If lost, they can deregister by scanning their passport and create a new agent.
{% endhint %}

## Ed25519

For agents that use Ed25519 keys natively (common in AI agent frameworks). No wallet required — the NFT owner is derived from the Ed25519 public key.

**Flow:**

1. Agent provides its Ed25519 public key
2. Agent signs a challenge with its Ed25519 private key
3. Scan passport with Self app — NFT is minted to an address derived from the Ed25519 pubkey
4. Deregister anytime by scanning passport again

**Use when:** Your agent framework uses Ed25519 keys (OpenClaw, Eliza, IronClaw) and you don't need a separate human wallet binding.

{% hint style="info" %}
The NFT owner address is deterministically derived from the Ed25519 public key. The agent can prove ownership of the NFT by signing with the same Ed25519 key.
{% endhint %}

## Ed25519-Linked

Combines Ed25519 agent keys with a human wallet for ownership. The human's wallet holds the NFT while the agent operates with Ed25519 keys.

**Flow:**

1. Connect browser wallet
2. Agent provides its Ed25519 public key
3. Agent signs a challenge with its Ed25519 private key
4. Scan passport with Self app — NFT is minted to the human's wallet, linked to the Ed25519 agent key
5. Human retains custody of the NFT while the agent operates independently

**Use when:** Your agent uses Ed25519 keys but you want the NFT ownership tied to a human wallet for oversight and management.

## Privy

A UI wrapper over the linked/wallet-free flow that uses Privy for social login authentication. Users sign in with OAuth providers instead of connecting a wallet.

**Flow:**

1. User authenticates via social login (Google, Twitter, email, etc.)
2. Privy creates an embedded wallet behind the scenes
3. Agent keypair is generated
4. Agent signs challenge; registration proceeds as linked mode
5. Scan passport with Self app

**Use when:** Your users prefer social login over wallet connections, or you want a familiar OAuth-style onboarding experience.

{% hint style="warning" %}
Privy is a UI convenience layer — the underlying contract interaction is identical to linked or wallet-free mode. The Privy-managed wallet holds the NFT on behalf of the user.
{% endhint %}

## Smart Wallet

A UI wrapper that uses a passkey (Face ID / fingerprint) to create a smart account. No MetaMask, no seed phrase.

**Flow:**

1. Create passkey via biometrics (creates a ZeroDev Kernel smart account)
2. Agent keypair is generated
3. Agent signs challenge; smart wallet is set as guardian
4. Scan passport with Self app
5. Revoke agent anytime with biometrics (gaslessly on Celo mainnet)

**Use when:** You want the simplest user experience with no seed phrases, no browser extensions, and gasless management via Pimlico.

{% hint style="info" %}
On Celo Sepolia (testnet), the smart wallet is counterfactual only — it deploys on first mainnet management action.
{% endhint %}

## Verification Configs

All modes support 6 verification configurations, combining age thresholds with OFAC sanctions screening:

| Config Index | Age Requirement | OFAC Screening |
|-------------|-----------------|----------------|
| 0 | None | Off |
| 1 | 18+ | Off |
| 2 | 21+ | Off |
| 3 | None | On |
| 4 | 18+ | On |
| 5 | 21+ | On |

The config is selected via the second character of `userDefinedData` during registration.

## ZK-Attested Credentials

During registration, users can optionally disclose verified claims that are stored on-chain as ZK-attested credentials:

- Nationality (ISO 3166 country code)
- Full name
- Date of birth
- Gender
- Issuing state
- OFAC sanctions clearance
- Age verification (18+ or 21+)

All disclosures are optional. Raw passport data never leaves the user's device.
