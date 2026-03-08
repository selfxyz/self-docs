---
description: Terminal-based agent registration and deregistration workflows
---

# CLI

Self Agent ID includes a cross-language CLI for registering and deregistering agents from the terminal. Available in TypeScript, Python, and Rust with identical command surfaces.

## Install

{% tabs %}
{% tab title="TypeScript" %}
```bash
npm install -g @selfxyz/agent-sdk
# or use npx:
npx @selfxyz/agent-sdk register init ...
```
{% endtab %}

{% tab title="Python" %}
```bash
pip install selfxyz-agent-sdk
# Then use:
self-agent register init ...
```
{% endtab %}

{% tab title="Rust" %}
```bash
cargo install self-agent-sdk
# Then use:
self-agent register init ...
```
{% endtab %}
{% endtabs %}

## Registration Flow

The CLI uses a **browser handoff** pattern: the terminal creates a session, generates a URL, and the user completes the Self proof in their browser.

### Step 1: Create Session

```bash
self-agent register init \
  --mode linked \
  --human-address 0xYourWalletAddress \
  --network mainnet \
  --out .self/session.json
```

**Modes:** `self-custody`, `linked`, `wallet-free`, `ed25519`, `ed25519-linked`, `smartwallet`

{% hint style="info" %}
The default network is **mainnet**, which requires a real passport scanned via the Self app. Use `--network testnet` for development — testnet also requires the Self app, but you can generate mock documents within the app instead of using a real passport.
{% endhint %}

### Step 2: Open Browser Handoff

```bash
self-agent register open --session .self/session.json
```

Opens the handoff URL in the default browser. The user scans the QR code with the Self app.

### Step 3: Wait for Completion

```bash
self-agent register wait --session .self/session.json
```

Polls the registration status until the Hub V2 callback confirms verification.

### Step 4: Check Status

```bash
self-agent register status --session .self/session.json
```

Returns the current session state (pending, verified, failed).

### Step 5: Export Credentials

```bash
self-agent register export --session .self/session.json
```

Outputs the agent address, agent key (bytes32), agent ID, and private key for use in your agent's environment.

## Deregistration Flow

```bash
# Create deregistration session
self-agent deregister init \
  --mode self-custody \
  --human-address 0xYourWalletAddress \
  --network mainnet \
  --out .self/session-deregister.json

# Open browser for Self proof
self-agent deregister open --session .self/session-deregister.json

# Wait for completion
self-agent deregister wait --session .self/session-deregister.json
```

## Ed25519 Registration

For agents that use Ed25519 keys instead of Ethereum wallets. Two modes are available:

### Standalone Ed25519

Register an agent identified solely by its Ed25519 public key:

```bash
self-agent register init \
  --mode ed25519 \
  --ed25519-pubkey <hex> \
  --ed25519-signature <hex> \
  --network mainnet \
  --out .self/session.json
```

### Ed25519 Linked to Human

Register an Ed25519 agent linked to a human's Ethereum address:

```bash
self-agent register init \
  --mode ed25519-linked \
  --ed25519-pubkey <hex> \
  --ed25519-signature <hex> \
  --human-address 0xYourWalletAddress \
  --network mainnet \
  --out .self/session.json
```

{% hint style="info" %}
The `--ed25519-signature` is a hex-encoded signature over the session challenge, proving ownership of the Ed25519 private key. The `--ed25519-pubkey` is the hex-encoded 32-byte public key.
{% endhint %}

After `init`, the remaining steps (`open`, `wait`, `status`, `export`) are identical to the standard registration flow.

## Agent-Guided Flow (Recommended)

For automated onboarding, your backend or agent runtime orchestrates the CLI commands and sends the handoff URL to the user:

1. Backend calls `register init` and stores session state
2. Backend calls `register open` and forwards URL to user UI
3. User completes browser proof flow
4. Backend runs `register wait` and records the returned lifecycle state

{% hint style="info" %}
The agent-guided flow is the recommended integration pattern for services that onboard users programmatically. The CLI handles all the complexity of session management and proof verification.
{% endhint %}

## Session Schema (v1)

The session file (`.self/session.json`) contains:

```json
{
  "version": 1,
  "mode": "linked",
  "network": "mainnet",
  "humanAddress": "0x...",
  "agentAddress": "0x...",
  "agentPrivateKey": "0x...",
  "sessionId": "uuid",
  "handoffUrl": "https://selfagentid.xyz/cli/register?session=...",
  "status": "pending",
  "createdAt": "2026-02-22T..."
}
```

## Network Flag

| Value | Chain | Notes |
|-------|-------|-------|
| `mainnet` (default) | Celo Mainnet (42220) | Real passports required |
| `testnet` | Celo Sepolia (11142220) | Mock documents only |
