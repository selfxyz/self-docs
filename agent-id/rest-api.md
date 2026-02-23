---
description: REST API endpoints for agent registration, queries, and discovery
---

# REST API

Self Agent ID exposes REST endpoints for registration workflows, agent queries, and A2A discovery. All endpoints are served from the web app deployment.

The full OpenAPI 3.1 spec is available at [`openapi.yaml`](https://github.com/selfxyz/self-agent-id/blob/main/openapi.yaml) in the main repository — import it into Postman or use it to generate clients.

For interactive documentation, visit [selfagentid.xyz/api-docs](https://selfagentid.xyz/api-docs).

## Public Query Endpoints

These endpoints require no authentication.

### Get Agent Info

```
GET /api/agent/info/{chainId}/{agentId}
```

Returns agent registration details, verification status, and credentials.

**Example response:**

```json
{
  "agentId": 5,
  "chainId": 11142220,
  "agentAddress": "0x83fa4380903fecb801F4e123835664973001ff00",
  "isVerified": true,
  "proofProvider": "0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4",
  "verificationStrength": 2,
  "strengthLabel": "Standard",
  "credentials": {
    "nationality": "GBR",
    "olderThan": 18,
    "ofac": [false, false, false]
  },
  "registeredAt": 1740000000,
  "network": "testnet"
}
```

### List Agents by Human Address

```
GET /api/agent/agents/{chainId}/{address}
```

Returns all agent IDs registered by a specific human wallet address.

**Example response:**

```json
{
  "agents": [5, 12, 37],
  "chainId": 11142220,
  "humanAddress": "0xabc..."
}
```

### Verify Agent Proof-of-Human

```
GET /api/agent/verify/{chainId}/{agentId}
```

Checks whether an agent has valid proof-of-human verification, the proof provider address, verification strength, and Sybil metrics.

**Example response:**

```json
{
  "agentId": 5,
  "isVerified": true,
  "proofProvider": "0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4",
  "strengthLabel": "Standard",
  "humanAgentCount": 1,
  "maxAgentsPerHuman": 1
}
```

### Get Agent Card

```
GET /api/cards/{chainId}/{agentId}
```

Returns the A2A-compatible agent identity card in JSON format.

### Get Reputation Score

```
GET /api/reputation/{chainId}/{agentId}
```

Returns the agent's verification strength score from the proof provider.

**Example response:**

```json
{
  "score": 2,
  "hasProof": true,
  "providerName": "Self Protocol",
  "proofType": "Standard"
}
```

### Get Verification Status

```
GET /api/verify-status/{chainId}/{agentId}
```

Returns real-time proof status and freshness.

**Example response:**

```json
{
  "verified": true,
  "proofType": "Standard",
  "registeredAtBlock": "12345678",
  "providerAddress": "0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4"
}
```

### A2A Discovery

```
GET /.well-known/a2a/{agentId}?chain={chainId}
```

Redirects to the agent card resolver. Compatible with A2A agent discovery protocols.

### Service Discovery

```
GET /.well-known/self-agent-id.json
```

Returns the service discovery document with API base URL, supported networks, registration modes, and capabilities.

**Example response:**

```json
{
  "name": "Self Agent ID",
  "version": "1.0",
  "apiBase": "https://selfagentid.xyz/api/agent",
  "networks": ["mainnet", "testnet"],
  "registrationModes": ["verified-wallet", "agent-identity", "wallet-free", "smart-wallet"],
  "capabilities": ["register", "deregister", "verify", "credentials", "agent-card", "a2a"],
  "sessionTtlMs": 1800000
}
```

## Registration Endpoints

Used by the dApp and CLI during registration flows. Sessions use encrypted tokens with a 30-minute TTL.

### Create Registration Session

```
POST /api/agent/register
```

Creates a new registration session. Returns session token, QR code data, deep link, and the generated agent address.

**Request body:**

```json
{
  "mode": "agent-identity",
  "network": "testnet",
  "humanAddress": "0xYourWalletAddress",
  "disclosures": {
    "minimumAge": 18,
    "ofac": true,
    "nationality": false,
    "name": false
  }
}
```

Modes: `verified-wallet`, `agent-identity`, `wallet-free`, `smart-wallet`. Networks: `mainnet`, `testnet`.

**Example response:**

```json
{
  "sessionToken": "enc_...",
  "deepLink": "selfapp://verify?scope=...",
  "qrData": "selfapp://verify?scope=...",
  "agentAddress": "0x83fa...ff00",
  "mode": "agent-identity",
  "network": "testnet"
}
```

### Poll Registration Status

```
GET /api/agent/register/status?token={sessionToken}
```

Polls for registration completion. Returns stage: `qr-ready`, `proof-received`, `completed`, or `failed`.

**Example response:**

```json
{
  "stage": "completed",
  "agentId": 42,
  "agentAddress": "0x83fa...ff00",
  "txHash": "0xabc...",
  "sessionToken": "enc_..."
}
```

### Get QR Code

```
GET /api/agent/register/qr?token={sessionToken}
```

Returns the QR code payload and deep link for the current session.

### Self App Callback

```
POST /api/agent/register/callback?token={sessionToken}
```

Webhook endpoint called by the Self app after the user scans the QR and submits a passport proof.

### Export Agent Private Key

```
GET /api/agent/register/export?token={sessionToken}
```

After registration completes, export the agent's private key. Only available for `agent-identity` and `wallet-free` modes.

**Example response:**

```json
{
  "privateKey": "0xdeadbeef...",
  "agentAddress": "0x83fa...ff00",
  "agentId": 42
}
```

## Deregistration Endpoints

### Create Deregistration Session

```
POST /api/agent/deregister
```

Creates a deregistration session for an existing agent. The human must re-prove identity to burn the agent NFT.

**Request body:**

```json
{
  "network": "testnet",
  "agentAddress": "0xAgentAddress",
  "disclosures": {
    "minimumAge": 18,
    "ofac": true
  }
}
```

### Poll Deregistration Status

```
GET /api/agent/deregister/status?token={sessionToken}
```

Returns current deregistration stage. Once completed, the agent NFT has been burned.

### Deregistration Callback

```
POST /api/agent/deregister/callback?token={sessionToken}
```

Webhook endpoint for the Self app after the user confirms deregistration.

## Demo Endpoints

Used by the live demo page for testing agent verification patterns. All demo endpoints require `x-self-agent-*` signed headers (produced by `SelfAgent.signRequest()`).

### Verify Agent (Service Pattern)

```
POST /api/demo/verify
```

Accepts a signed agent request, verifies signature and on-chain status, returns agent info and credentials.

### Agent-to-Agent

```
POST /api/demo/agent-to-agent
```

Demo agent endpoint that verifies the caller, checks `sameHuman()`, and signs its response. Response headers include the demo agent's own `x-self-agent-*` signature for mutual verification.

### Chain Verify (Meta-Transaction)

```
POST /api/demo/chain-verify
```

Relayer endpoint that submits EIP-712 meta-transactions to the `AgentDemoVerifier` contract. Rate limited to 3 per hour per human nullifier.

**Request body:**

```json
{
  "agentKey": "0x...",
  "nonce": "0",
  "deadline": 1740001800,
  "eip712Signature": "0x...",
  "networkId": "celo-sepolia"
}
```

### Census (Aggregated Stats)

```
POST /api/demo/census  — contribute credentials
GET  /api/demo/census  — read aggregate stats
```

Demonstrates agent-gated data contribution and reading.

### AI Chat

```
POST /api/demo/chat
```

AI agent chat endpoint backed by a LangChain agent. Signed requests are verified on-chain; unsigned requests are treated as anonymous.

**Request body:**

```json
{
  "query": "What is Self Agent ID?",
  "session_id": "user-123"
}
```

## AA Proxy Endpoints

Account-abstraction proxy for the smart-wallet registration mode. These endpoints proxy JSON-RPC calls to a Pimlico bundler/paymaster, shielding the API key from the client. Origin-restricted and rate-limited.

### Issue AA Proxy Token

```
POST /api/aa/token?chainId={chainId}
```

Issues a short-lived token required by the bundler and paymaster endpoints. Rate limited to 30 requests per minute per IP.

**Example response:**

```json
{
  "token": "eyJhbGciOi...",
  "expiresAt": 1740001800000
}
```

### Bundler Proxy

```
POST /api/aa/bundler?chainId={chainId}
```

Proxies ERC-4337 bundler JSON-RPC calls. Requires the `x-aa-proxy-token` header from the token endpoint.

**Allowed methods:** `eth_chainId`, `eth_supportedEntryPoints`, `eth_estimateUserOperationGas`, `eth_sendUserOperation`, `eth_getUserOperationReceipt`, `eth_getUserOperationByHash`, `eth_gasPrice`, `eth_maxPriorityFeePerGas`.

**Request body (JSON-RPC 2.0):**

```json
{
  "jsonrpc": "2.0",
  "method": "eth_sendUserOperation",
  "params": [...],
  "id": 1
}
```

### Paymaster Proxy

```
POST /api/aa/paymaster?chainId={chainId}
```

Proxies ERC-4337 paymaster JSON-RPC calls. Requires the `x-aa-proxy-token` header.

**Allowed methods:** `pm_sponsorUserOperation`, `pm_getPaymasterData`, `pm_getPaymasterStubData`, `eth_chainId`.

## Authentication

- **Public endpoints** (info, agents, verify, cards, reputation, verify-status, discovery): No authentication required
- **Demo endpoints**: Require `x-self-agent-*` headers (signed via SDK)
- **Registration endpoints**: Session-based with encrypted tokens, used by dApp/CLI
- **AA Proxy endpoints**: Origin-restricted + `x-aa-proxy-token` header (from `/api/aa/token`)

## Chain ID Parameter

All endpoints that accept a `chainId` parameter support:
- `42220` — Celo Mainnet
- `11142220` — Celo Sepolia Testnet

{% hint style="warning" %}
Use chain ID `11142220` for Celo Sepolia, not `44787` (deprecated Alfajores).
{% endhint %}

## Error Codes

All errors return `{ "error": "message" }` with the appropriate HTTP status.

| Code | Meaning |
|------|---------|
| 400  | Bad request — invalid parameters, missing fields, or wrong mode |
| 401  | Invalid or tampered session token / missing auth headers |
| 403  | Origin check failed or agent verification failed |
| 404  | Agent not found on-chain |
| 409  | Operation not available at current session stage / replay detected |
| 410  | Session expired (30-minute TTL) |
| 413  | Request too large (AA proxy, 200 KB limit) |
| 429  | Rate limit exceeded |
| 500  | Server error — RPC failure or configuration issue |
| 503  | Service not configured (bundler, paymaster, or LangChain) |
