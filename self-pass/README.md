---
icon: passport
---

# Self Pass

Privacy-preserving identity verification using zero-knowledge proofs.

## Overview

Self Pass enables developers to verify real-world identity attributes without exposing personal data. Users scan their passport, national ID, Aadhaar card, or KYC attestation with the Self app, which generates a zero-knowledge proof on-device. Applications can then verify specific attributes (age, nationality, sanctions status) without ever seeing the underlying document.

## Supported Documents

* **Passports** — NFC-enabled passports from 60+ countries
* **EU National ID Cards** — Chip-enabled European identity cards
* **Aadhaar** — India's national identity system
* **KYC Attestations** — Third-party KYC provider attestations

## Getting Started

Start with the [Quickstart](quickstart.md) guide, or fork the [workshop repo](https://github.com/selfxyz/workshop) for a working example.

## Integration Options

| Path | Description | Guide |
|------|-------------|-------|
| **Frontend SDK** | Display QR codes to request proofs from users | [QRCode SDK](frontend/qrcode-sdk.md) |
| **Backend SDK** | Verify proofs server-side on a Node.js backend | [Backend Integration](backend/basic-integration.md) |
| **Smart Contracts** | Verify proofs on-chain in a trustless manner | [Contract Integration](contracts/basic-integration.md) |
| **Mobile SDK** | Embed Self verification directly in a React Native app | [Mobile SDK](mobile-sdk/getting-started.md) |

## Examples

* [Airdrop Protection](contracts/airdrop-example.md) — Gate token distribution to verified humans
* [Happy Birthday](contracts/happy-birthday-example.md) — Age-gated smart contract example
* [Soul Bound Token](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/SelfPassportERC721.sol) — Mint SBTs for verified users
* [Cross Chain (LayerZero)](https://github.com/selfxyz/self-layerzero-example) — Cross-chain verification
* [Cross Chain (Hyperlane)](https://github.com/selfxyz/workshop/tree/hyperlane-example) — Cross-chain verification

## Resources

* [Workshop Video (ETHGlobal Buenos Aires)](https://www.loom.com/share/8a6d116a5f66415998a496f06fefdc23)
* [Self Builder Group](https://t.me/+d2TGsbkSDmgzODVi)
* [Celo Testnet Faucet](https://faucet.celo.org/celo-sepolia)
