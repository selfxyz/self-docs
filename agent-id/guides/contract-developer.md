---
description: Gate on-chain actions behind verified agent identity
---

# Gating Smart Contracts

This guide shows how to use the SelfAgentRegistry from Solidity to gate smart contract functions by proof-of-human agent identity.

## 1. Import the Registry Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ISelfAgentRegistry {
    // Agent verification
    function isVerifiedAgent(bytes32 agentKey) external view returns (bool);
    function getAgentId(bytes32 agentKey) external view returns (uint256);
    function getAgentCredentials(uint256 agentId) external view returns (AgentCredentials memory);

    // Proof-of-human (IERC8004ProofOfHuman)
    function hasHumanProof(uint256 agentId) external view returns (bool);
    function isProofFresh(uint256 agentId) external view returns (bool);
    function proofExpiresAt(uint256 agentId) external view returns (uint256);
    function getProofProvider(uint256 agentId) external view returns (address);

    // Sybil detection
    function sameHuman(uint256 agentIdA, uint256 agentIdB) external view returns (bool);
    function getAgentCountForHuman(uint256 nullifier) external view returns (uint256);
    function getHumanNullifier(uint256 agentId) external view returns (uint256);

    // ERC-8004 metadata
    function setAgentURI(uint256 agentId, string calldata newURI) external;
    function getMetadata(uint256 agentId, string memory key) external view returns (bytes memory);
    function setMetadata(uint256 agentId, string calldata key, bytes calldata value) external;

    struct AgentCredentials {
        string issuingState;
        string[] name;
        string idNumber;
        string nationality;
        string dateOfBirth;
        string gender;
        string expiryDate;
        uint256 olderThan;
        bool[3] ofac;
    }
}
```

Or import from the contracts package:

```solidity
import {ISelfAgentRegistryReader} from "self-agent-id/contracts/src/interfaces/ISelfAgentRegistryReader.sol";
```

## 2. `onlyVerifiedAgent` Modifier

The most common pattern — gate a function to verified agents only:

```solidity
contract MyContract {
    ISelfAgentRegistry public immutable registry;

    constructor(address _registry) {
        registry = ISelfAgentRegistry(_registry);
    }

    modifier onlyVerifiedAgent() {
        bytes32 agentKey = bytes32(uint256(uint160(msg.sender)));
        require(registry.isVerifiedAgent(agentKey), "Agent not human-verified");
        _;
    }

    function protectedAction() external onlyVerifiedAgent {
        // Only callable by verified agents
    }
}
```

The agent key derivation: `bytes32(uint256(uint160(address)))` — pads the 20-byte address to 32 bytes.

## 3. Query Credentials On-Chain

```solidity
function gatedByAge(uint256 minAge) external onlyVerifiedAgent {
    bytes32 agentKey = bytes32(uint256(uint160(msg.sender)));
    uint256 agentId = registry.getAgentId(agentKey);
    ISelfAgentRegistry.AgentCredentials memory creds = registry.getAgentCredentials(agentId);

    require(creds.olderThan >= minAge, "Age requirement not met");
    // creds.nationality, creds.ofac, creds.dateOfBirth, etc.
}
```

## 4. Check Proof Freshness

Proofs expire. Always check freshness for security-critical operations:

```solidity
function requireFreshProof(bytes32 agentKey) internal view {
    require(registry.isVerifiedAgent(agentKey), "Agent not verified");
    uint256 agentId = registry.getAgentId(agentKey);
    require(registry.isProofFresh(agentId), "Human proof expired — re-authentication required");
}
```

{% hint style="warning" %}
`isVerifiedAgent()` checks that the agent has a registered proof, but it does NOT check expiry. For time-sensitive operations, always call `isProofFresh()` as well.
{% endhint %}

## 5. Sybil Checks

Detect or prevent multiple agents from the same human:

```solidity
// Check if two agents are the same person
bool same = registry.sameHuman(agentIdA, agentIdB);

// Count how many agents this human has
uint256 nullifier = registry.getHumanNullifier(agentId);
uint256 count = registry.getAgentCountForHuman(nullifier);
require(count <= maxAllowed, "Too many agents for this human");
```

## 6. EIP-712 Meta-Transaction Pattern

For gasless agent verification, use a relayer that submits EIP-712 typed data on behalf of the agent:

```solidity
// Agent signs off-chain:
//   domain: { name, version, chainId, verifyingContract }
//   types:  { Verify: [agentKey, nonce, deadline] }
//   values: { agentKey, nonce, deadline }

function metaVerifyAgent(
    bytes32 agentKey,
    uint256 nonce,
    uint256 deadline,
    bytes calldata signature
) external returns (uint256 agentId) {
    require(block.timestamp <= deadline, "Expired");
    require(nonces[agentKey] == nonce, "Invalid nonce");

    // Recover signer from EIP-712 signature
    bytes32 structHash = keccak256(abi.encode(VERIFY_TYPEHASH, agentKey, nonce, deadline));
    bytes32 digest = _hashTypedDataV4(structHash);
    address signer = ECDSA.recover(digest, signature);

    // Verify signer matches agent key
    require(bytes32(uint256(uint160(signer))) == agentKey, "Signer mismatch");
    require(registry.isVerifiedAgent(agentKey), "Not verified");

    nonces[agentKey]++;
    agentId = registry.getAgentId(agentKey);
}
```

See `AgentDemoVerifier.sol` in the repo for a complete implementation.

## 7. Reference Contracts

The repo includes two demo contracts you can use as templates:

### AgentDemoVerifier

EIP-712 meta-transaction verifier. Demonstrates gasless verification where a relayer submits on behalf of agents.

- Source: [`contracts/src/AgentDemoVerifier.sol`](https://github.com/selfxyz/self-agent-id/blob/main/contracts/src/AgentDemoVerifier.sol)
- Key function: `metaVerifyAgent(agentKey, nonce, deadline, signature)`

### AgentGate

Access gate requiring age-verified agents. Demonstrates direct `msg.sender` verification with credential checks.

- Source: [`contracts/src/AgentGate.sol`](https://github.com/selfxyz/self-agent-id/blob/main/contracts/src/AgentGate.sol)
- Key functions: `checkAccess(agentKey)`, `gatedAction(agentKey)`

## 8. Contract Addresses

### Celo Mainnet (Chain ID: 42220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry (proxy) | `0xaC3DF9ABf80d0F5c020C06B04Cced27763355944` |
| SelfHumanProofProvider | `0x4b036aFD959B457A208F676cf44Ea3ef73Ea3E3d` |
| AgentDemoVerifier | `0xD8ec054FD869A762bC977AC328385142303c7def` |
| AgentGate | `0x26e05bF632fb5bACB665ab014240EAC1413dAE35` |
| Hub V2 | `0xe57F4773bd9c9d8b6Cd70431117d353298B9f5BF` |

### Celo Sepolia Testnet (Chain ID: 11142220)

| Contract | Address |
|----------|---------|
| SelfAgentRegistry (proxy) | `0x043DaCac8b0771DD5b444bCC88f2f8BBDBEdd379` |
| SelfHumanProofProvider | `0x5E61c3051Bf4115F90AacEAE6212bc419f8aBB6c` |
| AgentDemoVerifier | `0xc31BAe8f2d7FCd19f737876892f05d9bDB294241` |
| AgentGate | `0x86Af07e30Aa42367cbcA7f2B1764Be346598bbc2` |
| Hub V2 | `0x16ECBA51e18a4a7e61fdC417f0d47AFEeDfbed74` |

## 9. Foundry Setup

{% hint style="warning" %}
Hub V2 uses the `PUSH0` opcode. You must use `--evm-version cancun` for all Foundry commands.
{% endhint %}

```bash
forge build --evm-version cancun
forge test --evm-version cancun
```

Or add to `foundry.toml`:

```toml
[profile.default]
evm_version = "cancun"
```

## Next Steps

- [Build an agent to interact with your contract](agent-builder.md)
- [Verify agents in your API](service-operator.md)
- [Troubleshooting](../troubleshooting.md)
