---
description: Register, sign, and verify an agent using only the SDK and the on-chain registry, with no website
---

# Register Without the Web App

There is no consumer website for Self Agent ID. You render the passport-scan QR in your own app, the Self mobile app submits the proof on-chain, and the registry mints the agent. Signing and verification are on-chain too.

This page shows the full loop with the SDK: register, sign, verify. It needs no hosted service at all. If you prefer, you can also register through the [CLI](cli.md) or the [REST/A2A API](rest-api.md) at `https://agent-api.self.xyz`, which return the QR for you to display.

## What you need

- `npm install @selfxyz/agent-sdk @selfxyz/qrcode ethers`
- The Self mobile app on a phone. On testnet you can generate mock documents in the app, so no real passport is needed.
- A Celo RPC. Public defaults: `https://forno.celo.org` (mainnet), `https://forno.celo-sepolia.celo-testnet.org` (testnet).

| Network | Chain ID | Registry |
|---------|----------|----------|
| Celo Mainnet | `42220` | `0xaC3DF9ABf80d0F5c020C06B04Cced27763355944` |
| Celo Sepolia (testnet) | `11142220` | `0x043DaCac8b0771DD5b444bCC88f2f8BBDBEdd379` |

## 1. Register an agent

Render the QR in your own frontend. The proof goes from the Self app to the on-chain Hub, which mints the agent NFT into the registry. No backend is involved.

```tsx
import { useEffect, useState } from 'react';
import { SelfAppBuilder, SelfQRcodeWrapper } from '@selfxyz/qrcode';
import { signRegistrationChallenge, buildAdvancedRegisterUserDataAscii } from '@selfxyz/agent-sdk';
import { Wallet } from 'ethers';

const REGISTRY = '0xaC3DF9ABf80d0F5c020C06B04Cced27763355944'; // Celo mainnet

export function RegisterAgent({ humanAddress }: { humanAddress: string }) {
  const [selfApp, setSelfApp] = useState<any>(null);
  const [agentKey] = useState(() => Wallet.createRandom());

  useEffect(() => {
    (async () => {
      // The agent key signs a challenge proving it controls the key
      const sig = await signRegistrationChallenge(agentKey.privateKey, {
        humanIdentifier: humanAddress,
        chainId: 42220,
        registryAddress: REGISTRY,
        nonce: 0,
      });

      // Encode the registration payload (config index is derived from disclosures)
      const disclosures = { minimumAge: 18, ofac: true };
      const userDefinedData = buildAdvancedRegisterUserDataAscii({
        agentAddress: agentKey.address,
        signature: sig,
        disclosures,
      });

      // Build the Self verification request; the registry is the on-chain endpoint
      setSelfApp(
        new SelfAppBuilder({
          version: 2,
          appName: 'My Agent',
          scope: 'self-agent-id',
          endpoint: REGISTRY,
          endpointType: 'celo', // 'staging_celo' for testnet
          userId: humanAddress,
          userIdType: 'hex',
          userDefinedData,
          disclosures,
        }).build(),
      );
    })();
  }, [humanAddress]);

  return selfApp ? (
    <SelfQRcodeWrapper
      selfApp={selfApp}
      onSuccess={() => {
        // Minted. Save agentKey.privateKey securely — it is the agent's key.
      }}
      onError={() => console.error('verification failed')}
    />
  ) : (
    <p>Loading…</p>
  );
}
```

The user scans the QR with the Self app. On `onSuccess` the agent NFT is minted. **Save the agent private key.** This example uses `linked` mode; for other modes (wallet-free, Ed25519, etc.) see [Registration Modes](registration-modes.md).

{% hint style="info" %}
For a headless flow, use `getUniversalLink(selfApp)` from `@selfxyz/qrcode` to produce the deep link instead of the React component, and read the new agent from the registry with `getAgentId` / `isVerifiedAgent`.
{% endhint %}

## 2. Publish the agent's identity document (optional but recommended)

Agents minted this way start with a blank `agentURI`. Publish a JSON document so services and indexers can discover the agent, then point the agent at it.

```typescript
import { generateRegistrationJSON } from '@selfxyz/agent-sdk';

const doc = generateRegistrationJSON({
  name: 'My Agent',
  description: 'A human-backed AI agent',
  image: 'https://my-agent.example.com/avatar.png',
  services: [{ name: 'A2A', endpoint: 'https://my-agent.example.com/a2a', version: '1.0' }],
});
// Host doc over HTTPS, then call registry.setAgentURI(agentId, url)
```

## 3. Sign requests as the agent

```typescript
import { SelfAgent } from '@selfxyz/agent-sdk';

const agent = new SelfAgent({
  privateKey: process.env.AGENT_PRIVATE_KEY!,
  registryAddress: '0xaC3DF9ABf80d0F5c020C06B04Cced27763355944',
  rpcUrl: 'https://forno.celo.org',
});

const res = await agent.fetch('https://some-service.example.com/api/protected', {
  method: 'POST',
  body: JSON.stringify({ hello: 'world' }),
});
```

## 4. Verify the agent in a service

The verifier recovers the signer and checks the registry on-chain. No API, no shared secret.

```typescript
import { SelfAgentVerifier } from '@selfxyz/agent-sdk';

const verifier = new SelfAgentVerifier({
  registryAddress: '0xaC3DF9ABf80d0F5c020C06B04Cced27763355944',
  rpcUrl: 'https://forno.celo.org',
});

app.use('/api', verifier.auth()); // rejects requests from unverified agents
```

That is the whole product: register on-chain, sign, verify on-chain. Nothing here depends on a hosted Self website.

## Next steps

- [Registration Modes](registration-modes.md) — pick the key type and ownership model
- [SDK Integration](sdk-integration.md) — signing and verification in TypeScript, Python, and Rust
- [Verification Patterns](verification-patterns.md) — `sameHuman`, proof freshness, policy controls
- [Smart Contracts](smart-contracts.md) — gate a contract on verified agents
