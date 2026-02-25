---
description: Common issues and solutions for Self Agent ID integration
---

# Troubleshooting

## `userDefinedData` encoding fails

**This is the #1 integration mistake.**

The Self SDK passes `userDefinedData` as a **UTF-8 string**, not raw bytes. Each position uses the ASCII character — `'0'` (0x30), not `0x00`.

```solidity
// WRONG — raw byte value
bytes32 data = bytes32(uint256(0));

// CORRECT — ASCII character
bytes32 data = bytes32(bytes1(uint8(uint8(bytes1("0")))));
```

Use `bytes32(bytes1(uint8(x)))` for byte positioning in `bytes32`.

{% hint style="danger" %}
If your contract receives garbled `userDefinedData`, this is almost certainly the cause.
{% endhint %}

## "Agent not verified" when querying

**Possible causes:**

1. **Wrong chain ID** — Are you querying the right network? Mainnet is `42220`, testnet is `11142220`.
2. **Wrong registry address** — Each network has its own registry. See [Smart Contracts](smart-contracts.md) for addresses.
3. **Agent not yet registered** — Registration requires the Self app callback to complete. Poll with `self_check_registration` or the REST API status endpoint.
4. **Agent was deregistered** — The soulbound NFT was burned. Re-registration is needed.
5. **Proof expired** — The agent's human proof has passed its validity window. Check `isProofFresh(agentId)` — if it returns `false`, the agent must re-authenticate by scanning their passport again.

## Agent proof expired

If `isProofFresh()` returns `false` but `hasHumanProof()` returns `true`:

- The proof validity window has elapsed (default: 1 year or document expiry, whichever is sooner)
- The soulbound NFT is NOT burned — it remains as a historical record
- The agent must re-authenticate via the Self app to restore fresh status
- SDK verifiers automatically reject expired proofs with a descriptive error

## Registration callback never arrives

**Check these:**

1. **`SELF_ENDPOINT` type mismatch** — Use `celo` for mainnet, `staging_celo` for testnet. If set wrong, the Self app sends the callback to the wrong network.
2. **`.env.local` key must be lowercase** — `NEXT_PUBLIC_SELF_ENDPOINT` (not `NEXT_PUBLIC_Self_Endpoint`). Environment variable names are case-sensitive.
3. **Callback URL not reachable** — If running locally, ensure the callback port is accessible. The CLI binds to `127.0.0.1` by default.
4. **Session expired** — Sessions have a 30-minute TTL. Start a new registration if expired.

## Signature verification fails

**Common causes:**

1. **Timestamp drift** — The verifier rejects signatures older than 5 minutes (default). Ensure client and server clocks are in sync.
2. **Body encoding** — The signature covers `keccak256(timestamp + METHOD + path + bodyHash)`. If the body is modified in transit (e.g., by a proxy re-serializing JSON), verification fails.
3. **Path vs full URL** — The SDK signs the URL path (e.g., `/api/data`), not the full URL. Ensure your verifier extracts the path correctly.
4. **Wrong private key** — The signing key must match the registered agent address.

## Wrong chain ID

{% hint style="warning" %}
Celo Sepolia chain ID is **11142220**, not `44787` (deprecated Alfajores). This is a common mistake when migrating from older Celo testnet configurations.
{% endhint %}

| Network | Chain ID | RPC |
|---------|----------|-----|
| Celo Mainnet | `42220` | `https://forno.celo.org` |
| Celo Sepolia | `11142220` | `https://forno.celo-sepolia.celo-testnet.org` |

## Foundry build fails

If you see errors about `PUSH0` or unsupported opcodes:

```bash
# Hub V2 uses PUSH0 (EVM version cancun)
forge build --evm-version cancun
forge test --evm-version cancun
forge script ... --evm-version cancun
```

Add to `foundry.toml`:

```toml
[profile.default]
evm_version = "cancun"
```

## Provider verification untrusted

If agents pass `isVerifiedAgent()` but you don't trust the proof source:

```typescript
const verifier = SelfAgentVerifier.create()
  .requireSelfProvider()  // Ensures Self Protocol as provider
  .build();
```

Without `requireSelfProvider()`, any approved provider's proofs are accepted. Self Protocol's provider is the only one currently deployed, but this guard protects against future third-party providers.

## WebAuthn / passkey errors

**Firefox blocks WebAuthn on `http://localhost`**. Use one of:

- **Chrome** on `http://localhost` (Chrome allows it)
- **HTTPS** with a local cert (e.g., `mkcert`)
- **Deployed URL** with HTTPS

This only affects smart wallet mode (passkey-based registration).

## Smart wallet not gasless on testnet

Smart wallet mode is **counterfactual only** on testnet — the smart account address is computed but not deployed, so gasless transactions aren't available.

On **mainnet**, the Pimlico paymaster sponsors gas for registration transactions, enabling full gasless UX.

## Blockscout / Celoscan verification issues

- **Blockscout** does not require an API key for contract verification.
- **Celoscan** — use the **Sourcify verifier** (not the etherscan verifier). The Celoscan API endpoint is flaky; Sourcify is more reliable.

```bash
# Celoscan via Sourcify
forge verify-contract <address> <Contract> \
  --chain 42220 \
  --verifier sourcify
```

## MCP "no identity" error

If MCP tools that require identity (sign, fetch, get identity) fail:

- `SELF_AGENT_PRIVATE_KEY` is not set in your MCP config
- Query tools (`self_lookup_agent`, `self_verify_agent`, `self_list_agents_for_human`) work without a key
- Set the key in your MCP config `env` block to enable full mode

```json
{
  "mcpServers": {
    "self-agent-id": {
      "command": "npx",
      "args": ["@selfxyz/mcp-server"],
      "env": {
        "SELF_AGENT_PRIVATE_KEY": "0x...",
        "SELF_NETWORK": "mainnet"
      }
    }
  }
}
```
