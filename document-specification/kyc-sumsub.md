# KYC (Sumsub)

Self supports a KYC document type backed by the Sumsub verification flow.

## Overview

KYC is represented in verification payloads as attestation ID `4`.

In `@selfxyz/core`, this ID is currently exported as `SELFRICA_ID_CARD`:

```typescript
ATTESTATION_ID.SELFRICA_ID_CARD // 4
```

## User Flow

1. User selects `kyc` as document type in the Self app.
2. App starts the Sumsub flow using an access token from the TEE service.
3. After completion, verification is finalized asynchronously (websocket/push).
4. The app stores the resulting KYC attestation and can generate disclose proofs.

## Backend Verification

Backend verification uses the same `SelfBackendVerifier.verify(...)` API:

```typescript
const result = await verifier.verify(
  4,              // KYC / Sumsub
  proof,
  publicSignals,
  userContextData
);
```

You can allow all document types with `AllIds`, or allow KYC explicitly in your `allowedIds` map.

## Notes

* KYC is integrated into the same V2 proof and verification stack used by other document types.
* Existing disclosure fields and config checks (age, OFAC, country restrictions) continue to apply through the standard verifier interfaces.
