---
description: Tiered soulbound NFT rewarding AI agents for on-chain activity on Celo
---

# Celo Agent Visa

The Celo Agent Visa is a tiered soulbound NFT system that rewards AI agents for on-chain activity on the Celo blockchain. Agents progress through tiers based on transaction count and stablecoin volume, earning increasingly prestigious visa status.

## Tier System

| Tier | Name | Requirements | Manual Review |
|------|------|-------------|---------------|
| 1 | **Tourist Visa** | Self Agent Registry entry + 1 transaction | No |
| 2 | **Work Visa** | Human proof + 1,000 txs OR $5,000 stablecoin volume | No |
| 3 | **Citizenship** | Human proof + 10,000 txs OR $15,000 stablecoin volume | Yes |

Tier 2 and 3 use **OR** logic for metrics — meeting either the transaction count or volume threshold qualifies the agent. Tier 3 (Citizenship) requires manual review approval from the Celo team.

## How It Works

1. **Register an agent** via [Self Agent ID](overview.md) — this is a prerequisite
2. **Transact on Celo** — the scoring service monitors transaction count and stablecoin volume (USDT, USDC, cUSD)
3. **Check eligibility** — visit the visa dashboard or query the API
4. **Claim your visa** — the system mints a soulbound NFT at no gas cost (relayer pays)
5. **Progress through tiers** — as your metrics grow, request upgrades to higher tiers

All on-chain operations are gasless for end users. The Self relayer infrastructure covers transaction costs.

## Deployed Contracts

| Network | Contract | Address |
|---------|----------|---------|
| Celo Mainnet | CeloAgentVisa | `0xCa97f7586CF9De62B8ca516d7Ee25f6AEae5e109` |
| Celo Sepolia | CeloAgentVisa | `0xf049FD6260Fce964B82728A86CF1BbEB8AB3E875` |

The contract is a soulbound ERC-721 (non-transferable) using UUPS upgradeable pattern. Visa NFTs are permanently tied to the agent's identity and cannot be traded.

## API Endpoints

All visa endpoints are served from the Self Agent ID web app.

### Get Visa Status

```
GET /api/visa/{chainId}/{agentId}
```

Returns the agent's current tier, metrics, eligibility for each tier, and manual review status.

**Example response:**

```json
{
  "agentId": 42,
  "chainId": 42220,
  "tier": 1,
  "tierName": "Tourist Visa",
  "metrics": {
    "transactionCount": 1250,
    "volumeUsd": 6200,
    "lastUpdated": 1745123456
  },
  "eligibility": {
    "1": true,
    "2": true,
    "3": false
  },
  "reviewRequestedTier": 0,
  "manualReviewApproved": false
}
```

### List Agents by Wallet

```
GET /api/visa/agents?wallet={address}&chainId={chainId}
```

Returns all agent IDs owned by a wallet address.

### Claim or Upgrade Visa

```
POST /api/visa/claim
```

Gasless visa mint (for new visas) or tier upgrade. The relayer submits the transaction on behalf of the user.

**Request body:**

```json
{
  "chainId": "42220",
  "agentId": "42",
  "targetTier": 2,
  "agentWallet": "0x..."
}
```

Returns `422 REVIEW_REQUIRED` if the target tier requires manual review that hasn't been approved yet. Returns `422 NOT_ELIGIBLE` if metrics don't meet thresholds.

### Request Manual Review

```
POST /api/visa/request-review
```

Requests manual review from the Celo team for Tier 2 or 3 upgrades.

**Request body:**

```json
{
  "chainId": "42220",
  "agentId": "42",
  "targetTier": 2
}
```

## Reading Visa Status On-Chain

```typescript
import { getContract } from "viem";

const visa = getContract({
  address: "0xCa97f7586CF9De62B8ca516d7Ee25f6AEae5e109",
  abi: VISA_ABI,
  client: publicClient,
});

// Get current tier (0 = no visa, 1 = Tourist, 2 = Work, 3 = Citizenship)
const tier = await visa.read.getTier([BigInt(agentId)]);

// Get cached metrics
const metrics = await visa.read.getMetrics([BigInt(agentId)]);
// Returns: [transactionCount, volumeUsd, lastUpdated]

// Check eligibility for a specific tier
const eligible = await visa.read.checkTierEligibility([BigInt(agentId), 2]);
```

## Metrics & Scoring

Metrics are calculated off-chain by a scoring service and pushed on-chain periodically:

* **Transaction count** — all outgoing transactions from the agent's wallet (via RPC)
* **Stablecoin volume** — USD volume of ERC-20 transfers for supported stablecoins (via TheGraph subgraph)

**Supported stablecoins (Celo Mainnet):**

| Token | Address |
|-------|---------|
| USDT | `0x48065fbbE25f71C9282ddf5e1cD6D6A887483D5e` |
| USDC | `0xcebA9300f2b948710d2653dD7B07f33A8B32118C` |
| cUSD (USDm) | `0x765DE816845861e75A25fCA122bb6898B8B1282a` |

## Claim Flow

```
Agent registered in Self Agent Registry
        ↓
Agent transacts on Celo (transfers, swaps, contract calls)
        ↓
Scoring service detects activity and pushes metrics on-chain
        ↓
Agent checks eligibility via dashboard or API
        ↓
┌─ Tier 1 (Tourist): Claim directly — no review needed
├─ Tier 2 (Work): Claim directly once metrics are met
└─ Tier 3 (Citizenship): Request review → Celo team approves → Claim upgrade
        ↓
Soulbound NFT minted/upgraded (gasless)
```

{% hint style="info" %}
The visa contract uses `agentId` as its primary key, not wallet address. This means if an agent's wallet changes via the Self Agent Registry, the visa follows the agent identity.
{% endhint %}
