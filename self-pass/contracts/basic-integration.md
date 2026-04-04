---
description: >-
  This page explains how to integrate your smart contracts with Self’s on‑chain
  verification flow using the abstract base SelfVerificationRoot.
---

# Basic Integration

{% hint style="danger" %}
**Troubleshooting Celo Sepolia**: If you encounter a `Chain 11142220 not supported` error when deploying to Celo Sepolia, try to update Foundry to version 0.3.0:

```bash
foundryup --install 0.3.0
```
{% endhint %}

## Overview

The `@selfxyz/contracts` SDK provides you with a `SelfVerificationRoot` abstract contract that wires your contract to the Identity Verification Hub V2. Your contract receives a callback with disclosed, verified attributes only after the proof succeeds.

### Key flow

1. Your contract exposes `verifySelfProof(bytes proofPayload, bytes userContextData)` from the abstract contract.
2. It takes a verification config from your contract and forwards a packed input to Hub V2.
3. If the proof is valid, the Hub calls back your contract’s `onVerificationSuccess(bytes output, bytes userData)` .
4. You implement custom logic in `customVerificationHook(...)`.

## SelfVerificationRoot

This is an abstract contract that you must override by providing custom logic for returning a config id along with a hook that is called with the disclosed attributes. Here's what you need to override:

### 1. `getConfigId`

```solidity
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier,
    bytes memory userDefinedData
) public view virtual override returns (bytes32) 
```

Return the **verification config ID** that the hub should enforce for this request. In simple cases, you may store a single config ID in storage and return it. In advanced cases, compute a dynamic config id based on the inputs.

**Example (static config):**

```solidity
bytes32 public verificationConfigId;

function getConfigId(
    bytes32, bytes32, bytes memory
) public view override returns (bytes32) {
    return verificationConfigId;
}
```

### 2. `customVerificationHook`

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory userData
) internal virtual override
```

This is called **after** hub verification succeeds. Use it to:

* Mark the user as verified
* Mint/allowlist/gate features
* Emit events or write your own structs

## Constructor & Scope

```solidity
constructor(
    address hubV2, 
    string memory scopeSeed
) SelfVerificationRoot(hubV2, scopeSeed) {}
```

`SelfVerificationRoot` computes a **scope** at deploy time:

* It Poseidon‑hashes the **contract address** (chunked) with your **`scopeSeed`** to produce a unique `uint256` scope.
* The hub enforces that **submitted proofs match this scope**.

Why scope matters:

* Prevents cross‑contract proof replay.
* Allow anonymity between different applications as the nullifier is calculated as a function of the scope.

**Guidelines**

* Keep `scopeSeed` short (≤31 ASCII bytes). Example: `"proof-of-human"`.
* **Changing contract address changes the scope** (by design). Re‑deploys will need a fresh frontend config.
* You can read the current scope on‑chain via `function scope() public view returns (uint256)`.

{% hint style="info" %}
You can get the hub addresses from [deployed-contracts.md](deployed-contracts.md "mention")
{% endhint %}

## Setting Verification Configs

A verification config is simply what you want to verify your user against. Your contract must reference a **verification config** that the hub recognizes. Typical steps:

1. **Format and register** the config off‑chain or in a setup contract:

```solidity
SelfStructs.VerificationConfigV2 public verificationConfig;
bytes32 public verificationConfigId;

constructor(
    address hubV2, 
    string memory scopeSeed, 
    SelfUtils.UnformattedVerificationConfigV2 memory rawCfg
) SelfVerificationRoot(hubV2, scopeSeed) {
    // 1) Format the human‑readable struct into the on‑chain wire format
    verificationConfig = SelfUtils.formatVerificationConfigV2(rawCfg);

    // 2) Register the config in the Hub. **This call RETURNS the configId.**
    verificationConfigId = IIdentityVerificationHubV2(hubV2).setVerificationConfigV2(verificationConfig);
}
```

2. **Return the config id** from `getConfigId(...)` (static or dynamic):

```solidity
function getConfigId(
    bytes32, 
    bytes32, 
    bytes memory
) public view override returns (bytes32) {
    return verificationConfigId;
}
```

Here's how you would create a raw config:

```solidity
import { SelfUtils } from "@selfxyz/contracts/contracts/libraries/SelfUtils.sol";

// Inside your contract constructor or setup function:
string[] memory forbiddenCountries = new string[](1);
forbiddenCountries[0] = CountryCodes.UNITED_STATES; // ISO 3-letter codes, max 40 countries

SelfUtils.UnformattedVerificationConfigV2 memory verificationConfig = SelfUtils
    .UnformattedVerificationConfigV2({
        olderThan: 18,              // Minimum age (0 = no age check)
        forbiddenCountries: forbiddenCountries, // Countries to block (empty array = no restriction)
        ofacEnabled: false          // Enable OFAC sanctions screening
    });
```

{% hint style="warning" %}
Only a maximum of 40 countries are allowed!
{% endhint %}

### Frontend ↔ Contract config must match

{% hint style="danger" %}
The **frontend disclosure/verification config** used to produce the proof must **exactly match** the **contract’s verification config** (the `configId` you return). Otherwise the hub will detect a **mismatch** and verification fails.
{% endhint %}

Common pitfalls:

* Frontend uses `minimumAge: 18` but contract config expects `21` .
* Frontend uses different **scope** (e.g., points to a different contract address or uses a different `scopeSeed`).

{% hint style="success" %}
**Best practice:** Generate the config **once**, register it with the hub to get `configId`, and reference that same id in your dApp’s builder payload.
{% endhint %}

### Extracting data from a users proof

The `customVerificationHook` receives a `GenericDiscloseOutputV2` struct with all verified attributes. Here are all available fields:

| Field | Type | Description | Requires Disclosure |
|-------|------|-------------|:---:|
| `attestationId` | `bytes32` | Document type: `1` = Passport, `2` = EU ID Card, `3` = Aadhaar, `4` = KYC | No |
| `userIdentifier` | `uint256` | User's identifier. Derive address: `address(uint160(output.userIdentifier))` | No |
| `nullifier` | `uint256` | Unique per-user per-scope value for Sybil resistance | No |
| `forbiddenCountriesListPacked` | `uint256[4]` | Packed bitfield of excluded countries | No |
| `olderThan` | `uint256` | Minimum age verified (e.g. `18`). `0` if not checked | No |
| `ofac` | `bool[3]` | OFAC check results: `[0]` = passport number, `[1]` = name+DOB, `[2]` = name+YOB | No |
| `issuingState` | `string` | ISO 3-letter code of issuing country (e.g. `"GBR"`) | Yes |
| `name` | `string[]` | User's name fields from document | Yes |
| `idNumber` | `string` | Document number (passport number, ID number, etc.) | Yes |
| `nationality` | `string` | ISO 3-letter nationality code | Yes |
| `dateOfBirth` | `string` | Date of birth in document format | Yes |
| `gender` | `string` | Gender (`"M"` or `"F"`) | Yes |
| `expiryDate` | `string` | Document expiry date | Yes |

**"Requires Disclosure"** means the field is only populated if your frontend [disclosure config](../disclosures.md) explicitly requests it. Fields not requested will be empty/zero.

{% hint style="info" %}
The format of disclosed fields can vary by document type. Passports use ICAO MRZ format, Aadhaar uses different date and name formats, and KYC fields depend on the provider. See the [Document Specifications](../document-specification/aadhaar.md) section for per-document-type details.
{% endhint %}

**Example — extracting nationality and age in your hook:**

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory userData
) internal override {
    // Always available
    require(output.olderThan >= 18, "Must be 18+");
    
    // Only available if disclosure was requested
    string memory nationality = output.nationality; // e.g. "GBR"
    
    // Derive user's address
    address user = address(uint160(output.userIdentifier));
    
    // Check OFAC results (if enabled in config)
    // ofac[0] = passport number match, ofac[1] = name+DOB, ofac[2] = name+YOB
    // All must be false (not sanctioned) for verification to pass
}
```

The [Happy Birthday Example](happy-birthday-example.md) contains a full working example of extracting data from the `output` object.

## Minimal Example: Proof Of Human

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {SelfVerificationRoot} from "@selfxyz/contracts/contracts/abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts/interfaces/ISelfVerificationRoot.sol";
import {SelfStructs} from "@selfxyz/contracts/contracts/libraries/SelfStructs.sol";
import {SelfUtils} from "@selfxyz/contracts/contracts/libraries/SelfUtils.sol";
import {IIdentityVerificationHubV2} from "@selfxyz/contracts/contracts/interfaces/IIdentityVerificationHubV2.sol";

/**
 * @title ProofOfHuman
 * @notice Test implementation of SelfVerificationRoot for the docs
 * @dev This contract provides a concrete implementation of the abstract SelfVerificationRoot
 */
contract ProofOfHuman is SelfVerificationRoot {
    // Storage for testing purposes
    SelfStructs.VerificationConfigV2 public verificationConfig;
    bytes32 public verificationConfigId;

    // Events for testing
    event VerificationCompleted(
        ISelfVerificationRoot.GenericDiscloseOutputV2 output,
        bytes userData
    );

    /**
     * @notice Constructor for the test contract
     * @param identityVerificationHubV2Address The address of the Identity Verification Hub V2
     */
    constructor(
        address identityVerificationHubV2Address, // Hub V2 address — see Deployed Contracts page
        uint256 scopeSeed,                        // Unique app identifier (≤31 ASCII bytes), hashed into scope
        SelfUtils.UnformattedVerificationConfigV2 memory _verificationConfig // What to verify (age, countries, OFAC)
    ) SelfVerificationRoot(identityVerificationHubV2Address, scopeSeed) {
        verificationConfig = 
            SelfUtils.formatVerificationConfigV2(_verificationConfig);
        verificationConfigId = 
            IIdentityVerificationHubV2(identityVerificationHubV2Address)
            .setVerificationConfigV2(verificationConfig);
    }
    
    /**
     * @notice Implementation of customVerificationHook for testing
     * @dev This function is called by onVerificationSuccess after hub address validation
     * @param output The verification output from the hub
     * @param userData The user data passed through verification
     */
    function customVerificationHook(
        ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
        bytes memory userData
    ) internal override {
        emit VerificationCompleted(output, userData);
    }

    function getConfigId(
        bytes32 /* destinationChainId */,
        bytes32 /* userIdentifier */,
        bytes memory /* userDefinedData */
    ) public view override returns (bytes32) {
        return verificationConfigId;
    }
}
```
