# KYC

Self supports a KYC document type for identity verification.

## Overview

KYC is represented in verification payloads as attestation ID `4`.

## User flow

1. User selects `kyc` as document type in the Self app.
2. App starts the KYC verification flow using an access token from the TEE service.
3. After completion, verification is finalized asynchronously (websocket/push).
4. The app stores the resulting KYC attestation and can generate disclose proofs.

## What you receive

When verification completes against a flow that allows KYC, the `verification.completed` webhook fires with `proof_attributes` describing the disclosed predicates exactly as it would for any other document type.

To allow KYC for a flow, enable it under the **Documents** sub-tab of your flow's Configure view. See [Supported documents](../../flows/supported-documents.md) for the capability matrix.

## Notes

* KYC is integrated into the same V2 proof and verification stack used by other document types.
* Standard disclosure fields and config checks (age, OFAC, country restrictions) apply.
