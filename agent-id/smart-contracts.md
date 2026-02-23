---
description: On-chain contracts for agent registration, verification, and access control
---

# Smart Contracts

Self Agent ID deploys 7 contracts on Celo.

## Contract Overview

| Contract | Role |
|----------|------|
| **SelfAgentRegistry** | Core registry — ERC-721 soulbound NFTs, registration modes, credentials, sybil detection |
| **SelfHumanProofProvider** | Connects to Self Protocol Hub V2, verifies ZK proofs, manages verification configs |
| **AgentDemoVerifier** | EIP-712 meta-transaction verifier for gasless on-chain agent verification |
| **AgentGate** | Abstract contract with `onlyVerifiedAgent` modifier for gating access |
| **SelfReputationProvider** | Verification strength scoring from proof providers |
| **SelfValidationProvider** | Real-time proof status and freshness checks |
| **LocalRegistryHarness** | Local mock for testing without Hub V2 dependency |

## Deployed Addresses

### Celo Mainnet (42220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry | [`0x62e37d0f6c5f67784b8828b3df68bcdbb2e55095`](https://celoscan.io/address/0x62e37d0f6c5f67784b8828b3df68bcdbb2e55095) |
| SelfHumanProofProvider | [`0x0B43f87aE9F2AE2a50b3698573B614fc6643A084`](https://celoscan.io/address/0x0B43f87aE9F2AE2a50b3698573B614fc6643A084) |
| AgentDemoVerifier | [`0x063c3bc21F0C4A6c51A84B1dA6de6510508E4F1e`](https://celoscan.io/address/0x063c3bc21F0C4A6c51A84B1dA6de6510508E4F1e) |
| AgentGate | [`0x2d710190e018fCf006E38eEB869b25C5F7d82424`](https://celoscan.io/address/0x2d710190e018fCf006E38eEB869b25C5F7d82424) |

### Celo Sepolia (11142220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry | [`0x42cea1b318557ade212bed74fc3c7f06ec52bd5b`](https://celo-sepolia.blockscout.com/address/0x42cea1b318557ade212bed74fc3c7f06ec52bd5b) |
| SelfHumanProofProvider | [`0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4`](https://celo-sepolia.blockscout.com/address/0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4) |
| AgentDemoVerifier | [`0x26e05bF632fb5bACB665ab014240EAC1413dAE35`](https://celo-sepolia.blockscout.com/address/0x26e05bF632fb5bACB665ab014240EAC1413dAE35) |
| AgentGate | [`0x71a025e0e338EAbcB45154F8b8CA50b41e7A0577`](https://celo-sepolia.blockscout.com/address/0x71a025e0e338EAbcB45154F8b8CA50b41e7A0577) |

## Key Interfaces

### Reading Agent State

```solidity
// Check if an agent is verified
bool verified = registry.isVerifiedAgent(agentKey);

// Get agent ID from key
uint256 agentId = registry.getAgentId(agentKey);

// Get proof provider
address provider = registry.agentProofProvider(agentId);

// Get ZK-attested credentials
AgentCredentials memory creds = registry.getAgentCredentials(agentId);
```

### Sybil Detection

```solidity
// Check if two agents share the same human
bool same = registry.sameHuman(agentId1, agentId2);

// Count agents for a nullifier
uint256 count = registry.getAgentCountForHuman(nullifier);
```

### Gating Access

Use `AgentGate` as a base contract:

```solidity
import { AgentGate } from "./AgentGate.sol";

contract MyProtocol is AgentGate {
    constructor(address _registry) AgentGate(_registry) {}

    function protectedAction() external onlyVerifiedAgent {
        // Only verified agents can call this
        bytes32 agentKey = bytes32(uint256(uint160(msg.sender)));
        uint256 agentId = registry.getAgentId(agentKey);
        // ... your logic
    }
}
```

### EIP-712 Meta-Transactions

The `AgentDemoVerifier` contract accepts EIP-712 signed messages for gasless verification:

```solidity
// Domain: { name: "AgentDemoVerifier", version: "1", chainId, verifyingContract }
// Type: MetaVerify(bytes32 agentKey, uint256 nonce, uint256 deadline)

function metaVerify(
    bytes32 agentKey,
    uint256 nonce,
    uint256 deadline,
    bytes calldata signature
) external;
```

## Deregistration

Three paths to deregister an agent (all burn the soulbound NFT):

1. **Hub V2 callback** — `_deregisterAgent()` triggered by a new deregistration proof
2. **Self-deregister** — `selfDeregister(agentId)` called by the NFT owner
3. **Guardian revoke** — `guardianRevoke(agentId)` called by the agent's guardian

All converge on `_revokeAgent()` which clears all state (credentials, nullifier, provider, guardian, metadata) and burns the NFT.

## Integration with Self Protocol

The `SelfHumanProofProvider` connects to Self Protocol's [Identity Verification Hub V2](../contract-integration/basic-integration.md). See the [Working with userDefinedData](../contract-integration/working-with-userdefineddata.md) guide for details on how the registration data is encoded.
