---
description: How agents prove their identity to services, other agents, and smart contracts
---

# Verification Patterns

Self Agent ID supports four verification patterns, each suited to different integration scenarios.

## Pattern 1: Agent-to-Service

The most common pattern. An agent signs HTTP requests; a service verifies the signature and checks the on-chain registry.

```
Agent                          Service
  │                               │
  │── POST /api (signed) ────────▶│
  │   headers:                    │
  │     x-self-agent-address      │── ecrecover signer
  │     x-self-agent-signature    │── derive agentKey
  │     x-self-agent-timestamp    │── isVerifiedAgent(key)
  │                               │── check credentials
  │◀── 200 OK ────────────────────│
```

**SDK support:** `SelfAgent.fetch()` on agent side, `SelfAgentVerifier.auth()` middleware on service side.

## Pattern 2: Agent-to-Agent (Peer Verification)

Two agents verify each other's identity. Both sign their requests, and both verify the other's signature against the registry.

```
Agent A                        Agent B
  │                               │
  │── POST (signed by A) ────────▶│
  │                               │── verify A's signature
  │                               │── sameHuman(A, B) check
  │◀── Response (signed by B) ────│
  │── verify B's signature        │
```

**Key feature:** `sameHuman(agentIdA, agentIdB)` detects whether two agents share the same human backer without revealing who that human is.

## Pattern 3: Agent-to-Chain (Direct)

The agent calls a smart contract directly using `msg.sender`. The contract checks the registry.

```solidity
import { AgentGate } from "./AgentGate.sol";

contract MyProtocol is AgentGate {
    constructor(address registry) AgentGate(registry) {}

    function protectedAction() external onlyVerifiedAgent {
        // Only registered, human-backed agents can call this
    }
}
```

**Best for:** Verified Wallet mode where the agent's wallet address IS the `msg.sender`.

## Pattern 4: Agent-to-Chain (Meta-Transaction)

The agent signs an EIP-712 typed data message; a relayer submits the transaction on-chain. The contract verifies the EIP-712 signature matches a registered agent.

```
Agent                    Relayer                  Contract
  │                         │                        │
  │── sign EIP-712 ──────── │                        │
  │   (agentKey, nonce,     │── metaVerify(sig) ────▶│
  │    deadline)            │                        │── ecrecover signer
  │                         │                        │── isVerifiedAgent()
  │                         │◀── tx confirmed ───────│
  │◀── receipt ─────────────│                        │
```

**Best for:** Gasless verification where agents don't hold native tokens. The `AgentDemoVerifier` contract implements this pattern.

## Sybil Resistance

Services can enforce their own sybil policies using three approaches:

### Strict (max 1 agent per human)

```typescript
const verifier = SelfAgentVerifier.create()
  .maxAgentsPerHuman(1)
  .build();
```

### Moderate (allow N agents)

```typescript
const verifier = SelfAgentVerifier.create()
  .maxAgentsPerHuman(5)
  .build();
```

### Detection Only

```typescript
// Allow unlimited, but check relationships
const verifier = SelfAgentVerifier.create()
  .maxAgentsPerHuman(0) // unlimited
  .build();

// Then use sameHuman() for analytics
const areSame = await registry.sameHuman(agentId1, agentId2);
```

## Credential-Based Verification

Verifiers can require specific ZK-attested credentials:

```typescript
const verifier = SelfAgentVerifier.create()
  .requireAge(18)           // Agent's human must be 18+
  .requireOFAC()            // Agent's human must be OFAC-cleared
  .requireNationality("US") // Specific nationality
  .build();
```

All credential checks are verified against on-chain ZK-attested data — no additional identity check needed.
