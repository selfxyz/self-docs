---
description: On-chain contracts for agent registration, verification, and access control
---

# Smart Contracts

Self Agent ID deploys 9 contracts on Celo. The core registry implements `IERC8004` and the `IERC8004ProofOfHuman` extension.

## Contract Overview

| Contract | Role |
|----------|------|
| **SelfAgentRegistry** | Core registry — ERC-721 soulbound NFTs, IERC8004 + IERC8004ProofOfHuman, registration modes, credentials, proof expiry, sybil detection (UUPS upgradeable) |
| **SelfHumanProofProvider** | Implements `IHumanProofProvider` — metadata wrapper connecting to Self Protocol Hub V2 (verification strength: 100) |
| **SelfReputationRegistry** | ERC-8004 Reputation Registry — aggregated feedback with document-type weighted signals (UUPS upgradeable) |
| **SelfValidationRegistry** | ERC-8004 Validation Registry — on-demand validation requests/responses from third parties (UUPS upgradeable) |
| **AgentDemoVerifier** | EIP-712 meta-transaction verifier for gasless on-chain agent verification |
| **AgentGate** | Abstract contract with `onlyVerifiedAgent` modifier for gating access |
| **LocalRegistryHarness** | Local mock for testing without Hub V2 dependency |

## Deployed Addresses

### Celo Mainnet (42220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry (proxy) | [`0xaC3DF9ABf80d0F5c020C06B04Cced27763355944`](https://celoscan.io/address/0xaC3DF9ABf80d0F5c020C06B04Cced27763355944) |
| SelfHumanProofProvider | [`0x4b036aFD959B457A208F676cf44Ea3ef73Ea3E3d`](https://celoscan.io/address/0x4b036aFD959B457A208F676cf44Ea3ef73Ea3E3d) |
| SelfReputationRegistry (proxy) | [`0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4`](https://celoscan.io/address/0x69Da18CF4Ac27121FD99cEB06e38c3DC78F363f4) |
| SelfValidationRegistry (proxy) | [`0x71a025e0e338EAbcB45154F8b8CA50b41e7A0577`](https://celoscan.io/address/0x71a025e0e338EAbcB45154F8b8CA50b41e7A0577) |
| AgentDemoVerifier | [`0xD8ec054FD869A762bC977AC328385142303c7def`](https://celoscan.io/address/0xD8ec054FD869A762bC977AC328385142303c7def) |
| AgentGate | [`0x26e05bF632fb5bACB665ab014240EAC1413dAE35`](https://celoscan.io/address/0x26e05bF632fb5bACB665ab014240EAC1413dAE35) |
| Hub V2 | [`0xe57F4773bd9c9d8b6Cd70431117d353298B9f5BF`](https://celoscan.io/address/0xe57F4773bd9c9d8b6Cd70431117d353298B9f5BF) |

### Celo Sepolia (11142220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry (proxy) | [`0x043DaCac8b0771DD5b444bCC88f2f8BBDBEdd379`](https://celo-sepolia.blockscout.com/address/0x043DaCac8b0771DD5b444bCC88f2f8BBDBEdd379) |
| SelfHumanProofProvider | [`0x5E61c3051Bf4115F90AacEAE6212bc419f8aBB6c`](https://celo-sepolia.blockscout.com/address/0x5E61c3051Bf4115F90AacEAE6212bc419f8aBB6c) |
| SelfReputationRegistry (proxy) | [`0x3Bb0A898C1C0918763afC22ff624131b8F420CC2`](https://celo-sepolia.blockscout.com/address/0x3Bb0A898C1C0918763afC22ff624131b8F420CC2) |
| SelfValidationRegistry (proxy) | [`0x84cA20B8A1559F136dA03913dbe6A7F68B6B240B`](https://celo-sepolia.blockscout.com/address/0x84cA20B8A1559F136dA03913dbe6A7F68B6B240B) |
| AgentDemoVerifier | [`0xc31BAe8f2d7FCd19f737876892f05d9bDB294241`](https://celo-sepolia.blockscout.com/address/0xc31BAe8f2d7FCd19f737876892f05d9bDB294241) |
| AgentGate | [`0x86Af07e30Aa42367cbcA7f2B1764Be346598bbc2`](https://celo-sepolia.blockscout.com/address/0x86Af07e30Aa42367cbcA7f2B1764Be346598bbc2) |
| Hub V2 | [`0x16ECBA51e18a4a7e61fdC417f0d47AFEeDfbed74`](https://celo-sepolia.blockscout.com/address/0x16ECBA51e18a4a7e61fdC417f0d47AFEeDfbed74) |

## ERC-8004 Interfaces

The registry implements two standard interfaces:

### IERC8004 (Base Identity Registry)

```solidity
// Registration (3 overloads)
function register() external returns (uint256 agentId);
function register(string calldata agentURI) external returns (uint256 agentId);
function register(string calldata agentURI, string[] calldata keys, bytes[] calldata values) external returns (uint256 agentId);

// Agent URI
function setAgentURI(uint256 agentId, string calldata newURI) external;

// Metadata (key-value store)
function getMetadata(uint256 agentId, string memory key) external view returns (bytes memory);
function setMetadata(uint256 agentId, string calldata key, bytes calldata value) external;

// Agent Wallet (EIP-712 signature required from newWallet)
function setAgentWallet(uint256 agentId, address newWallet, uint256 deadline, bytes calldata signature) external;
function getAgentWallet(uint256 agentId) external view returns (address);
function unsetAgentWallet(uint256 agentId) external;
```

### IERC8004ProofOfHuman (Extension)

```solidity
// Register with human proof from approved provider
function registerWithHumanProof(string calldata agentURI, address proofProvider, bytes calldata proof, bytes calldata providerData) external returns (uint256 agentId);

// Revoke proof (requires re-proving same human)
function revokeHumanProof(uint256 agentId, address proofProvider, bytes calldata proof, bytes calldata providerData) external;

// Proof queries
function hasHumanProof(uint256 agentId) external view returns (bool);
function proofExpiresAt(uint256 agentId) external view returns (uint256);
function isProofFresh(uint256 agentId) external view returns (bool);
function getHumanNullifier(uint256 agentId) external view returns (uint256);
function getProofProvider(uint256 agentId) external view returns (address);
function isApprovedProvider(address provider) external view returns (bool);

// Sybil detection
function sameHuman(uint256 agentIdA, uint256 agentIdB) external view returns (bool);
function getAgentCountForHuman(uint256 nullifier) external view returns (uint256);
```

## Key Interfaces

### Reading Agent State

```solidity
// Check if an agent is verified
bool verified = registry.isVerifiedAgent(agentKey);

// Get agent ID from key
uint256 agentId = registry.getAgentId(agentKey);

// Get proof provider
address provider = registry.getProofProvider(agentId);

// Get ZK-attested credentials
AgentCredentials memory creds = registry.getAgentCredentials(agentId);

// Check proof freshness and expiry
bool fresh = registry.isProofFresh(agentId);
uint256 expiresAt = registry.proofExpiresAt(agentId);
```

### Sybil Detection

```solidity
// Check if two agents share the same human
bool same = registry.sameHuman(agentId1, agentId2);

// Count agents for a nullifier
uint256 count = registry.getAgentCountForHuman(nullifier);
```

### Metadata & Agent URI

```solidity
// Set/get agent URI (ERC-8004)
registry.setAgentURI(agentId, "https://example.com/agent.json");

// Set/get metadata key-value pairs
registry.setMetadata(agentId, "name", abi.encode("My Agent"));
bytes memory value = registry.getMetadata(agentId, "name");

// Set agent wallet (requires EIP-712 signature from newWallet)
registry.setAgentWallet(agentId, newWallet, deadline, signature);
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

## Proof Expiry

Each agent's human proof has a validity window. After expiry, `isProofFresh()` returns `false` and the agent must re-authenticate.

```solidity
// Check if proof is still valid
bool fresh = registry.isProofFresh(agentId);

// Get the expiry timestamp
uint256 expiresAt = registry.proofExpiresAt(agentId);
```

{% hint style="info" %}
Expired proofs do **not** burn the soulbound NFT. The NFT remains as a historical record, but `hasHumanProof()` returns `true` while `isProofFresh()` returns `false`. The agent must re-authenticate to restore fresh status.
{% endhint %}

The expiry is calculated as the minimum of:
- Document expiry date (from passport)
- Registration timestamp + `maxProofAge` (default: 1 year)

## Reputation Registry

The `SelfReputationRegistry` is a standalone ERC-8004 Reputation Registry scoped to the `SelfAgentRegistry`. It stores aggregated feedback from clients with document-type weighted signals.

```solidity
// Give feedback on an agent
reputationRegistry.giveFeedback(agentId, value, decimals, tag1, tag2, endpoint, uri, hash);

// Agent responds to feedback
reputationRegistry.appendResponse(agentId, clientAddress, feedbackIndex, uri, hash);

// Get aggregated summary
(uint256 count, int256 sum, uint8 decimals) = reputationRegistry.getSummary(agentId, clients, tag1, tag2);
```

**Document-type weights** (auto-assigned at registration):

| Document | Weight | Tag |
|----------|--------|-----|
| E-Passport (NFC) | 100 | `passport-nfc` |
| EU ID Card (NFC) | 100 | `id-card-nfc` |
| Aadhaar | 80 | `aadhaar` |
| KYC | 50 | `kyc` |

## Validation Registry

The `SelfValidationRegistry` is a standalone ERC-8004 Validation Registry for on-demand validation requests.

```solidity
// Request validation from a third party
bytes32 requestHash = validationRegistry.requestValidation(agentId, validator, requestURI);

// Validator responds
validationRegistry.respondValidation(requestHash, response, responseURI, tag);

// Get validation summary
(uint256 count, uint256 avgResponse) = validationRegistry.getSummary(agentId, validators, tag);
```

## Deregistration

Three paths to deregister an agent (all burn the soulbound NFT):

1. **Hub V2 callback** — `_deregisterAgent()` triggered by a new deregistration proof
2. **Self-deregister** — `selfDeregister(agentId)` called by the NFT owner
3. **Guardian revoke** — `guardianRevoke(agentId)` called by the agent's guardian

All converge on `_revokeAgent()` which clears all state (credentials, nullifier, provider, guardian, metadata) and burns the NFT.

## Governance

The registry uses role-based governance with two multisig wallets:

| Role | Capability |
|------|-----------|
| `SECURITY_ROLE` | Approve proof providers, set verification configs, set max proof age |
| `OPERATIONS_ROLE` | Grant roles, link reputation/validation registries |

## Integration with Self Protocol

The `SelfHumanProofProvider` connects to Self Protocol's [Identity Verification Hub V2](../contract-integration/basic-integration.md). See the [Working with userDefinedData](../contract-integration/working-with-userdefineddata.md) guide for details on how the registration data is encoded.

All core registries (SelfAgentRegistry, SelfReputationRegistry, SelfValidationRegistry) are deployed as UUPS proxies with ERC-7201 namespaced storage, enabling safe upgrades without storage collision.
