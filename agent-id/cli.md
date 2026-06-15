---
description: Terminal-based agent registration and deregistration workflows
---

# CLI

Self Agent ID includes a cross-language CLI for registering and deregistering agents from the terminal. Available in TypeScript, Python, and Rust with identical command surfaces.

{% hint style="info" %}
The CLI talks to the API at `https://agent-api.self.xyz`. Override it with the `SELF_AGENT_API_BASE` environment variable if you run your own deployment. The CLI fetches the scannable QR from the API (`GET /api/qr/{sessionToken}`), so no consumer web app is needed.
{% endhint %}

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

**Modes:** `linked`, `wallet-free`, `ed25519`, `ed25519-linked`, `smartwallet`

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
  --mode linked \
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

## Canonical challenge domain

For every mode except `self-custody`, the agent key signs a challenge proving it controls the key. All SDKs hash the same domain so a session created by one CLI is verifiable by any other:

```
keccak256(abi.encodePacked("self-agent-id:register:", humanIdentifier, chainId, registryAddress, nonce))
```

The hashing and the `(r, s, v)` signature split must match across TypeScript, Python, and Rust. The same value is reproduced on-chain when the registry verifies the agent signature alongside the Self ZK proof.

## Session schema (v1)

The session file (`.self/session.json`) is a structured record, not a flat blob. Top-level keys:

| Key | Notes |
|-----|-------|
| `version` | Schema version (`1`) |
| `operation` | `register` or `deregister` |
| `sessionId`, `createdAt`, `expiresAt` | Session identity and TTL |
| `mode`, `disclosures` | Registration mode and selected disclosures |
| `network` | `{ chainId, rpcUrl, registryAddress, endpointType, appUrl, appName, scope }` |
| `registration` | `{ humanIdentifier, agentAddress, userDefinedData, challengeHash, signature, smartWalletTemplate? }` (`challengeHash`/`signature` for non-`self-custody` modes) |
| `callback` | `{ listenHost: "127.0.0.1", listenPort, path: "/callback", stateToken, used, lastStatus?, lastError? }` |
| `state` | `{ stage, updatedAt, lastError?, agentId?, guardianAddress? }` |
| `secrets` | `{ agentPrivateKey }` — generated-key modes only (`linked`, `wallet-free`, `smartwallet`) |

### Local session stages

The local session file moves through these `state.stage` values:

```
initialized → handoff_opened → callback_received → onchain_verified
                                                  → onchain_deregistered   (deregister flow)
                                                  → failed | expired
```

{% hint style="info" %}
These are the **local CLI** stages. They are distinct from the **API** registration stages (`qr-ready`, `proof-received`, `completed`, `failed`) returned by `GET /api/agent/register/status`. The CLI reconciles the API/on-chain state into its own session file.
{% endhint %}

## Browser handoff & callback contract

`register open` encodes the session into a `payload=<base64url(json)>` parameter for the API's `/cli/register` handoff page. The payload carries: `version`, `operation`, `sessionId`, `stateToken`, `callbackUrl`, `mode`, `chainId`, `registryAddress`, `endpointType`, `appName`, `scope`, `humanIdentifier`, `expectedAgentAddress`, `expiresAt`, and optionally `disclosures`, `userDefinedData`, `smartWalletTemplate`.

When the browser flow completes, it POSTs JSON back to the CLI's loopback callback (`http://127.0.0.1:<port>/callback`): `{ sessionId, stateToken, status: "success" | "error", timestamp, operation?, error?, guardianAddress? }`. The CLI rejects callbacks whose `sessionId` / `stateToken` do not match, and rejects replays.

## Security

1. Exporting the agent private key is blocked unless `--unsafe` is passed explicitly.
2. Session and key files use restricted file permissions. Treat them as sensitive local state.
3. The callback listener binds to the loopback host only.
4. Session expiry is enforced before handoff and wait operations.
5. Rotate or delete old session files after a successful registration.

## Refreshing an expired proof

Human proofs expire at `min(document expiry, registration time + maxProofAge)` (default `maxProofAge` ≈ 365 days). After expiry, `isProofFresh(agentId)` returns `false`. The CLI surfaces `proofExpiresAt` in `register status` and warns when expiry is within 30 days.

There is no in-place CLI refresh command. To refresh, run the full deregister flow then a new register flow, which mints a **new** `agentId` (update any stored references):

```bash
self-agent deregister init --mode linked --human-address 0x... --agent-address 0x... --network mainnet --out .self/dereg.json
self-agent deregister open --session .self/dereg.json    # complete Self proof
self-agent deregister wait --session .self/dereg.json

self-agent register init --mode linked --human-address 0x... --network mainnet --out .self/refresh.json
self-agent register open --session .self/refresh.json     # complete Self proof
self-agent register wait --session .self/refresh.json
```

{% hint style="info" %}
The REST API also exposes an in-place [refresh endpoint](rest-api.md#proof-refresh-endpoints) (`POST /api/agent/refresh`) that re-proves an existing agent without minting a new ID. The CLI does not wrap it yet.
{% endhint %}

## Network Flag

| Value | Chain | Notes |
|-------|-------|-------|
| `mainnet` (default) | Celo Mainnet (42220) | Real passports required |
| `testnet` | Celo Sepolia (11142220) | Mock documents only |
