---
description: Four ways to register an AI agent with proof-of-human
---

# Registration Modes

Self Agent ID supports four registration modes. All produce the same on-chain result — a verified, sybil-resistant agent NFT — but they differ in who holds the agent's private key and how the human manages their agent.

## Mode Comparison

| Mode | Wallet Required? | NFT Owner | Agent Key | Guardian | Best For |
|------|-----------------|-----------|-----------|----------|----------|
| **Verified Wallet** | Yes (MetaMask, etc.) | Human's wallet | Wallet address | N/A | On-chain gating, single agent |
| **Agent Identity** | Yes (registration only) | Human's wallet | Independent keypair | N/A | Autonomous agents, key rotation |
| **No Wallet** | No | Agent's address | Independent keypair | Optional | Non-crypto users |
| **Smart Wallet** | No (passkey) | Agent's address | Independent keypair | Passkey smart wallet | Simplest UX, gasless management |

## Verified Wallet

The human's wallet address becomes the agent key. No extra keypair to manage.

**Flow:**
1. Connect browser wallet (MetaMask, etc.)
2. Scan passport with Self app — ZK proof binds wallet address to human nullifier
3. Agent key is derived as `bytes32(uint256(uint160(walletAddress)))` inside the contract callback

**Use when:** You want the simplest setup and your wallet IS the agent (DAOs, token gating, DeFi access).

## Agent Identity (Recommended)

The agent generates its own dedicated keypair. The human's wallet is used only during registration.

**Flow:**
1. Connect browser wallet
2. A fresh agent keypair is generated in the browser
3. Agent signs a challenge proving key ownership (ECDSA)
4. Scan passport with Self app — contract verifies both ZK proof and agent signature in one step
5. Human's wallet key is never exposed to agent software

**Use when:** You're building autonomous agents that need their own identity, or you want key rotation without re-registering the human.

## No Wallet

No crypto wallet required. The agent generates a keypair and owns its own NFT.

**Flow:**
1. Agent keypair generated in browser
2. Agent signs a challenge proving key ownership
3. Scan passport with Self app — NFT is minted to the agent's own address
4. Deregister anytime by scanning passport again (ZK proof links to your unique identity)

**Use when:** The user has no crypto wallet and just needs a quick registration with their passport.

{% hint style="info" %}
In wallet-free mode, the user must save the agent's private key securely. If lost, they can deregister by scanning their passport and create a new agent.
{% endhint %}

## Smart Wallet

A passkey (Face ID / fingerprint) creates a Kernel smart account as guardian. No MetaMask, no seed phrase.

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
