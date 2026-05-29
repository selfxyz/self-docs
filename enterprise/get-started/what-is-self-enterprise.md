# What is Self Enterprise

Self Enterprise is the managed offering on top of the [Self protocol](https://docs.self.xyz). It bundles the parts you'd otherwise have to build:

| Concern | Open-source Self | Self Enterprise |
| --- | --- | --- |
| Configure a verification flow | Encode disclosures in your contract / backend | Configure in the dashboard, publish a version |
| Issue credentials to users | Run your own backend + hosted page | Hand the user a `verificationUrl` from the API |
| Verify a proof | Run Groth16 verifier yourself (or on-chain) | We verify on the hot path; you receive a signed webhook |
| Store the result | Build your own store + audit log | Optional decentralized storage; full audit log in the dashboard |
| Bill the work | n/a | Usage-based, metered via Metronome, paid via Stripe |

If your team wants a fast path from "we want ZK identity verification" to a working integration, Enterprise is the path. If your team prefers to own the full stack, the open-source SDK at [docs.self.xyz](https://docs.self.xyz) is still the right tool.

## Who is it for?

* **Consumer apps** running sybil resistance, age gates, or geographic compliance — anyone who needs verified identity but doesn't want to build verification infrastructure.
* **Fintech / regulated platforms** that need KYC-grade verification with auditable trails.
* **Marketplaces** that want to verify both sides of a trade without holding PII.
* **Internal compliance teams** at protocols who'd otherwise hand-roll a verifier.

## How it fits together

```
┌──────────────┐    sessions.create()    ┌──────────────┐
│ Your backend │ ─────────────────────▶  │  Self        │
└──────────────┘                          └──────┬───────┘
       ▲                                          │
       │ webhook (verification.completed)         │ hosted
       │                                          ▼
┌──────────────┐                          ┌──────────────┐
│ Your backend │  ◀──────────────────────  │  Self app   │
└──────────────┘    Svix-signed           │  (user)      │
                                          └──────────────┘
```

1. **You configure a flow** in the dashboard (what to verify — age, nationality, sanctions, etc.).
2. **You call `sessions.create(...)`** via the Enterprise SDK when a user needs to verify. We return a `verificationUrl`.
3. **Your user opens the URL** in their Self app and produces a ZK proof of the requested attributes.
4. **We verify the proof** on the hot path and fire a webhook to your backend.
5. **You read the verified attributes** from the webhook and proceed.

Next: [Quickstart](quickstart.md) walks the end-to-end integration in ten minutes.
