# Verification in the IdentityVerificationHub

The IdentityVerificationHub V2 is the core verification engine that processes zero-knowledge proofs and executes identity verification workflows.

## How the Hub Works

The Hub operates as a central verification coordinator that:

* **Receives Verification Requests** from contracts implementing `ISelfVerificationRoot`
* **Processes ZK Proofs** using specialized circuit verifiers
* **Applies Verification Rules** based on stored configurations
* **Returns Structured Results** to the calling contract

## Complete Verification Flow

### 1. Request Initiation

```solidity
// User contract calls Hub with proof data
function verifySelfProof(bytes calldata proofPayload, bytes calldata userContextData) external;
```

**What happens:**

#### TEE Proof Generation (Gas-Free for Users)

* **User Input**: User scans passport/ID with mobile app
* **TEE Processing**: Trusted Execution Environment securely processes identity data
* **ZK Proof Creation**: TEE generates zero-knowledge proof without revealing raw identity data

#### Relayer (Sponsored Transactions)

* **Proof Relay**: Relayer receives ZK proof from TEE
* **Gas Sponsorship**: Relayer pays all transaction gas fees on behalf of user
* **Onchain Submission**: Relayer submits proof to user's contract via blockchain transaction
* **User Experience**: User gets verified identity without any crypto/gas requirements

#### Contract Processing

* Contract receives ZK proof from TEE/Relayer
* Calls Hub's `verifySelfProof` with proof + user context data
* Hub begins processing the verification request

### 2. Input Decoding & Context Processing

```solidity
// Hub internal: _decodeInput() and _decodeUserContextData()
(HubInputHeader memory header, bytes calldata proofData) = _decodeInput(baseVerificationInput);
(configId, destChainId, userIdentifier, remainingData) = _decodeUserContextData(userContextData);
```

**What happens:**

* **Header Extraction**: Gets contract version, scope, attestation ID
* **Context Parsing**: Extracts config ID, destination chain, user identifier
* **Data Preparation**: Prepares proof data for verification

### 3. Configuration Retrieval

```solidity
// Hub loads verification configuration by configId
VerificationConfigV2 memory config = $v2._v2VerificationConfigs[configId];
```

**What happens:**

* Hub looks up stored verification configuration using configId
* Configuration contains all verification rules (age, countries, OFAC, etc.)
* If config doesn't exist, verification fails

### 4. Document Type Detection & Routing

```solidity
// Based on attestationId in header
if (attestationId == AttestationId.E_PASSPORT) {
    // Route to passport verification logic
} else if (attestationId == AttestationId.EU_ID_CARD) {
    // Route to EU ID card verification logic
}
```

**What happens:**

* Hub identifies document type from attestation ID
* Routes to appropriate verification pipeline
* Uses document-specific circuit verifiers

### 5. Basic Verification (\_basicVerification)

**Purpose:** Validates the **cryptographic correctness** and **security** of the ZK proof itself.

```solidity
// Hub performs 4 core verification steps
bytes memory proofOutput = _basicVerification(
    header,
    vcAndDiscloseProof,
    userContextData,
    userIdentifier
);
```

**What happens (4 verification scopes):**

#### Scope 1: Contract Validation

* **Scope Check**: Ensures proof was generated for the correct contract
* **User Identifier Check**: Validates user identity consistency

#### Scope 2: Registry & Timestamp Validation

* **Root Check**: Validates against Merkle tree in identity registry
* **Current Date Check**: Ensures proof is within valid time window

#### Scope 3: Cryptographic Proof Verification

* **Groth16 Proof Verification**: Validates ZK proof using circuit verifier
* **Public Signals Validation**: Verifies proof inputs/outputs match expectations

#### Scope 4: Raw Data Extraction

* **Output Generation**: Creates `PassportOutput` or `EuIdOutput` with raw field data
* **Data Preparation**: Prepares extracted data for business logic verification

**Output:** Raw identity data (`PassportOutput`/`EuIdOutput`) ready for custom verification.

### 6. Custom Verification (CustomVerifier.customVerify)

**Purpose:** Applies **business logic rules** and validates **identity attributes** against configuration requirements.

```solidity
// Hub applies custom verification logic
GenericDiscloseOutputV2 memory output = CustomVerifier.customVerify(
    header.attestationId,
    config,
    proofOutput
);
```

**What happens:**

* **Document Type Routing**: Routes to passport vs ID card specific verification
* **Business Rule Application**: Applies age, geographic, and sanctions requirements
* **Identity Data Extraction**: Converts raw data to structured, human-readable format
* **Final Validation**: Ensures all configuration requirements are met

**Output:** Structured identity data (`GenericDiscloseOutputV2`) with verification results.

#### Age Verification (olderThanEnabled)

```solidity
if (verificationConfig.olderThanEnabled) {
    if (!CircuitAttributeHandlerV2.compareOlderThan(
        attestationId, 
        passportOutput.revealedDataPacked, 
        verificationConfig.olderThan
    )) {
        revert InvalidOlderThan();
    }
}
```

* Validates user meets minimum age requirement
* Uses circuit-extracted age data for verification
* Example: config requires 18+, user is 20 → ✅ passes
* Example: config requires 21+, user is 18 → ❌ fails

#### Geographic Restrictions (forbiddenCountriesEnabled)

```solidity
if (verificationConfig.forbiddenCountriesEnabled) {
    for (uint256 i = 0; i < 4; i++) {
        if (passportOutput.forbiddenCountriesListPacked[i] != 
            verificationConfig.forbiddenCountriesListPacked[i]) {
            revert InvalidForbiddenCountries();
        }
    }
}
```

* Validates forbidden countries list matches exactly
* Uses packed representation for gas efficiency (4 uint256 array)
* Order in config must match proof's forbidden countries list

#### OFAC Sanctions Verification

```solidity
if (verificationConfig.ofacEnabled[0] || 
    verificationConfig.ofacEnabled[1] || 
    verificationConfig.ofacEnabled[2]) {
    if (!CircuitAttributeHandlerV2.compareOfac(
        attestationId,
        passportOutput.revealedDataPacked,
        verificationConfig.ofacEnabled[0], // passport number
        verificationConfig.ofacEnabled[1], // name + DOB
        verificationConfig.ofacEnabled[2]  // name + YOB
    )) {
        revert InvalidOfacCheck();
    }
}
```

* **Mode 0**: OFAC check using passport number
* **Mode 1**: OFAC check using name + date of birth
* **Mode 2**: OFAC check using name + year of birth
* Each mode can be independently enabled/disabled
* Uses circuit-provided OFAC verification results

### 7. Output Formatting & Generation

```solidity
// Hub formats verification results into structured output
output = _formatVerificationOutput(header.contractVersion, genericDiscloseOutput);
```

**What happens:**

* Raw proof signals converted to human-readable data
* Structured identity information extracted
* Verification results (age, OFAC, etc.) included

### 8. Result Delivery

```solidity
// Hub calls back to the original contract
ISelfVerificationRoot(callingContract).onVerificationSuccess(
    abi.encode(output), 
    userData
);
```

**What happens:**

* Hub calls `onVerificationSuccess` on the requesting contract
* Passes structured output + user-defined data
* Contract can then execute its custom business logic

## Data Structures

### VerificationConfigV2

```solidity
struct VerificationConfigV2 {
    bool olderThanEnabled;                    // Enable age verification
    uint256 olderThan;                        // Minimum age requirement  
    bool forbiddenCountriesEnabled;           // Enable country restrictions
    uint256[4] forbiddenCountriesListPacked;  // Packed forbidden countries
    bool[3] ofacEnabled;                      // OFAC verification modes
}
```

### GenericDiscloseOutputV2 (Verification Result)

```solidity
struct GenericDiscloseOutputV2 {
    bytes32 attestationId;                    // E_PASSPORT, EU_ID_CARD, AARDHAAR or SELFRICA_ID_CARD
    uint256 userIdentifier;                   // User's unique identifier
    uint256 nullifier;                        // Anti-replay nullifier
    uint256[4] forbiddenCountriesListPacked;  // Forbidden countries used
    
    // Disclosed identity information
    string issuingState;                      // Document issuing country
    string[] name;                            // [first, middle, last] names
    string idNumber;                          // Passport/ID number
    string nationality;                       // User's nationality
    string dateOfBirth;                       // Birth date (DD-MM-YY)
    string gender;                            // User's gender
    string expiryDate;                        // Document expiry date
    
    // Verification results
    uint256 olderThan;                        // Verified minimum age
    bool[3] ofac;                             // OFAC results [passport number, name+dob, name+yob]
}
```

## Key V2 Improvements

### Multi-Document Support

* Automatic routing for attestation types: E\_PASSPORT, EU\_ID\_CARD, AARDHAAR and SELFRICA\_ID\_CARD
* Document-specific verification pipelines
* Unified interface for different document types

### Structured Output

* Rich `GenericDiscloseOutputV2` with pre-extracted attributes
* No more manual parsing of raw field elements
* Type-safe access to identity data

### Flexible Configuration Management

* Reusable `VerificationConfigV2` stored in Hub
* Create configurations without contract redeployment
* Use [Self Configuration Tools](https://tools.self.xyz/) to register configs

### Enhanced Security

* Separate verification logic for each document type
* Improved timestamp and replay protection
* Gas-optimized verification checks
