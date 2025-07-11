# SelfBackendVerifier

The `SelfBackendVerifier` class is the backend component that validates zero-knowledge proofs generated by the Self mobile app. It acts as the "proof validator" in your verification flow, ensuring that users have provided valid identity document information according to your requirements.

## How Backend Verification Works

When a user scans your QR code and provides their identity document information, here's what happens on the backend:

### 1. Proof Reception
- Self mobile app sends a zero-knowledge proof to your API endpoint
- The proof contains cryptographic evidence that the user meets your requirements
- **No actual document data is transmitted** - only mathematical proofs

### 2. Verification Process
- **SelfBackendVerifier** validates the proof's cryptographic integrity
- Checks that the proof was generated for your specific application (scope)
- Verifies the proof against on-chain merkle roots (ensures document validity)
- Confirms all requirements (age, nationality, etc.) are met
- Returns detailed verification results

### 3. Configuration Management
- Uses **IConfigStorage** to determine verification requirements
- Supports different rules based on user context (via `userDefinedData`)
- Enables dynamic verification scenarios (e.g., different age requirements per transaction type)

## Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Mobile App    │    │   Self Relayer  │    │  Your Backend   │
│                 │    │                 │    │                 │
│ Generates ZK    │───▶│ Forwards proof  │───▶│ SelfBackend     │
│ Proof from      │    │ to your         │    │ Verifier        │
│ Document NFC    │    │ endpoint        │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
                                               ┌─────────────────┐
                                               │ IConfigStorage  │
                                               │                 │
                                               │ Determines      │
                                               │ verification    │
                                               │ requirements    │
                                               └─────────────────┘
```

## Constructor

```typescript
new SelfBackendVerifier(
  scope: string,
  endpoint: string,
  mockPassport: boolean,
  allowedIds: Map<AttestationId, boolean>,
  configStorage: IConfigStorage,
  userIdentifierType: UserIdType
)
```

### Parameters

| Parameter          | Type                          | Description                                                                                                |
| ------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------- |
| scope              | `string`                      | Your application's unique identifier. Must match the scope used in SelfAppBuilder. Max 31 characters.     |
| endpoint           | `string`                      | Your backend verification endpoint URL. Must be publicly accessible and match your frontend configuration. |
| mockPassport       | `boolean`                     | `false` for real documents (mainnet), `true` for testing with mock documents (testnet)                    |
| allowedIds         | `Map<AttestationId, boolean>` | Map of allowed document types. Key: attestation ID, Value: allowed status                                  |
| configStorage      | `IConfigStorage`              | Configuration storage implementation that determines verification requirements                               |
| userIdentifierType | `UserIdType`                  | Type of user identifier: `'uuid'` or `'hex'` (for blockchain addresses)                                    |

### Understanding the Parameters

#### Scope Matching
The `scope` parameter **must exactly match** the scope used in your `SelfAppBuilder`:
```typescript
// Frontend
const selfApp = new SelfAppBuilder({
  scope: "myapp-prod",  // This scope...
  // ... other config
});

// Backend
const verifier = new SelfBackendVerifier(
  "myapp-prod",  // ...must match this scope
  // ... other params
);
```

#### Document Types (AttestationId)
Control which document types your application accepts:
- `1` - Electronic passports
- `2` - EU ID cards

```typescript
// Accept all supported document types
import { AllIds } from '@selfxyz/core';
const verifier = new SelfBackendVerifier(
  // ... other params
  AllIds,  // Accepts all supported document types
  // ... other params
);

// Accept only specific document types
const allowedIds = new Map<AttestationId, boolean>();
allowedIds.set(1, true);  // Electronic passports only
const verifier = new SelfBackendVerifier(
  // ... other params
  allowedIds,
  // ... other params
);
```

#### Network Selection
The `mockPassport` parameter determines which blockchain network to use:
- `false` - **Production**: Celo Mainnet with real document data
- `true` - **Testing**: Celo Alfajores testnet with mock document data

## Configuration Storage (IConfigStorage)

The `IConfigStorage` interface is crucial for dynamic verification requirements. It determines what to verify based on user context.

### Built-in Storage Classes

#### DefaultConfigStore - Simple Static Configuration
Perfect for applications with uniform verification requirements:

```typescript
import { DefaultConfigStore } from '@selfxyz/core';

const configStore = new DefaultConfigStore({
  minimumAge: 21,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
});

// Always returns the same configuration regardless of user or context
```

#### InMemoryConfigStore - Dynamic Configuration
Enables different verification rules based on user context:

```typescript
import { InMemoryConfigStore } from '@selfxyz/core';

const configStore = new InMemoryConfigStore(
  async (userIdentifier: string, userDefinedData: string) => {
    // Parse user-defined data to determine requirements
    const context = JSON.parse(userDefinedData);
    
    // Return different config IDs based on context
    if (context.action === 'high_value_transfer') {
      return 'strict_verification';
    } else if (context.action === 'basic_signup') {
      return 'standard_verification';
    }
    return 'default_verification';
  }
);

// Set up different verification configurations
await configStore.setConfig('strict_verification', {
  minimumAge: 21,
  excludedCountries: ['IRN', 'PRK', 'CUB'],
  ofac: true
});

await configStore.setConfig('standard_verification', {
  minimumAge: 18,
  excludedCountries: [],
  ofac: false
});
```

## Main Verification Method

### verify()

This is the core method that validates zero-knowledge proofs:

```typescript
async verify(
  attestationId: AttestationId,
  proof: VcAndDiscloseProof,
  pubSignals: BigNumberish[],
  userContextData: string
): Promise<VerificationResult>
```

#### Parameters

| Parameter       | Type                 | Description                                                       |
| --------------- | -------------------- | ----------------------------------------------------------------- |
| attestationId   | `AttestationId`      | Document type identifier (1 = electronic passport, 2 = EU ID card)           |
| proof           | `VcAndDiscloseProof` | Zero-knowledge proof object containing cryptographic proof arrays |
| pubSignals      | `BigNumberish[]`     | Public signals from the zero-knowledge proof                      |
| userContextData | `string`             | Hex-encoded string containing user context and configuration data |

#### Return Value

The method returns a `VerificationResult` object with comprehensive verification details:

```typescript
{
  attestationId: AttestationId;           // Document type that was verified
  isValidDetails: {
    isValid: boolean;                     // Overall cryptographic proof validity
    isOlderThanValid: boolean;            // Age requirement validation
    isOfacValid: boolean;                 // OFAC sanctions check result
  };
  forbiddenCountriesList: string[];      // Countries excluded from the proof
  discloseOutput: {                       // Disclosed document information
    nullifier: string;                    // Unique proof identifier (prevents reuse)
    forbiddenCountriesListPacked: string[];
    issuingState: string;                 // Country that issued the document
    name: string;                         // Full name (if disclosed)
    idNumber: string;                     // Document number
    nationality: string;                  // Nationality
    dateOfBirth: string;                  // Date of birth (if disclosed)
    gender: string;                       // Gender
    expiryDate: string;                   // Document expiry date
    olderThan: string;                    // Age verification result
    ofac: boolean[];                      // OFAC check results [passportNo, nameAndDob, nameAndYob]
  };
  userData: {
    userIdentifier: string;               // User identifier from context
    userDefinedData: string;              // Custom user data
  };
}
```

#### Error Handling

The method throws `ConfigMismatchError` when verification requirements don't match:

```typescript
try {
  const result = await verifier.verify(attestationId, proof, pubSignals, userContextData);
  // Handle successful verification
} catch (error: any) {
  if (error.name === 'ConfigMismatchError') {
    console.error('Configuration mismatches:', error.issues);
    // error.issues contains detailed information about what failed
  } else {
    console.error('Verification error:', error);
  }
}
```

**Common ConfigMismatch Types:**
- `InvalidId` - Attestation ID not in allowedIds
- `InvalidScope` - Proof was generated for a different application
- `InvalidRoot` - Merkle root not found on blockchain
- `InvalidForbiddenCountriesList` - Countries don't match configuration
- `InvalidMinimumAge` - Age requirement mismatch
- `InvalidTimestamp` - Proof timestamp out of valid range (±1 day)
- `InvalidOfac` - OFAC check requirements mismatch
- `ConfigNotFound` - Configuration not found in storage

## Complete Implementation Example

Here's a full example showing how to integrate `SelfBackendVerifier` in a real application:

### Setting Up the Verifier

```typescript
import { 
  SelfBackendVerifier,
  InMemoryConfigStore,
  AllIds,
  AttestationId
} from '@selfxyz/core';

// Configure dynamic verification based on user context
const configStorage = new InMemoryConfigStore(
  async (userIdentifier: string, userDefinedData: string) => {
    const context = JSON.parse(userDefinedData);
    
    switch (context.action) {
      case 'financial_transaction':
        return context.amount > 10000 ? 'high_value_kyc' : 'standard_kyc';
      case 'age_verification':
        return 'age_check_only';
      case 'geographic_restriction':
        return 'country_filter';
      default:
        return 'basic_verification';
    }
  }
);

// Set up different verification configurations
await configStorage.setConfig('high_value_kyc', {
  minimumAge: 21,
  excludedCountries: ['IRN', 'PRK', 'CUB'],
  ofac: true
});

await configStorage.setConfig('standard_kyc', {
  minimumAge: 18,
  excludedCountries: ['IRN', 'PRK'],
  ofac: false
});

await configStorage.setConfig('age_check_only', {
  minimumAge: 18,
  excludedCountries: [],
  ofac: false
});

await configStorage.setConfig('country_filter', {
  minimumAge: 0,
  excludedCountries: ['IRN', 'PRK', 'CUB'],
  ofac: false
});

// Initialize the verifier
const verifier = new SelfBackendVerifier(
  'myapp-prod',                    // Must match frontend scope
  'https://api.myapp.com/verify',  // Your verification endpoint
  false,                           // Production mode (real passports)
  AllIds,                          // Accept all document types
  configStorage,                   // Dynamic configuration
  'uuid'                           // User identifier type
);
```

### API Endpoint Implementation

```typescript
import { NextRequest, NextResponse } from "next/server";
import { countries, Country3LetterCode, SelfAppDisclosureConfig } from "@selfxyz/common";
import {
  countryCodes,
  SelfBackendVerifier,
  AllIds,
  DefaultConfigStore,
  VerificationConfig
} from "@selfxyz/core";

export async function POST(req: NextRequest) {
  console.log("Received request");
  try {
    const { attestationId, proof, publicSignals, userContextData } = await req.json();

    if (!proof || !publicSignals || !attestationId || !userContextData) {
      return NextResponse.json({
        status: 'error',
        result: false,
        reason: "Proof, publicSignals, attestationId and userContextData are required",
        error_code: "INVALID_INPUTS"
      }, { status: 200 });
    }

    const disclosures_config: VerificationConfig = {
      excludedCountries: [],
      ofac: false,
      minimumAge: 18,
    };

    const configStore = new DefaultConfigStore(disclosures_config);

    const selfBackendVerifier = new SelfBackendVerifier(
      "self-workshop",
      process.env.NEXT_PUBLIC_SELF_ENDPOINT || "",
      true,
      AllIds,
      configStore,
      "hex",
    );

    const result = await selfBackendVerifier.verify(
      attestationId,
      proof,
      publicSignals,
      userContextData
    );
    
    if (!result.isValidDetails.isValid) {
      return NextResponse.json({
        status: "error",
        result: false,
        reason: "Verification failed",
        error_code: "VERIFICATION_FAILED"
        details: result.isValidDetails,
      }, { status: 200 });
    }

    const saveOptions = (await configStore.getConfig(
      result.userData.userIdentifier
    )) as unknown as SelfAppDisclosureConfig;

    if (result.isValidDetails.isValid) {
      console.log(result.discloseOutput);

      return NextResponse.json({
        status: "success",
        result: result.isValidDetails.isValid,
        credentialSubject: result.discloseOutput,
        verificationOptions: {
          minimumAge: saveOptions.minimumAge,
          ofac: saveOptions.ofac,
          excludedCountries: saveOptions.excludedCountries?.map(
            (countryName) => {
              const entry = Object.entries(countryCodes).find(
                ([_, name]) => name === countryName
              );
              return entry ? entry[0] : countryName;
            }
          ),
        },
      });
    } else {
      return NextResponse.json({
        status: "error",
        result: result.isValidDetails.isValid,
        reason: "Verification failed",
        error_code: "VERIFICATION_FAILED"
        details: result,
      }, { status: 200 });
    }
  } catch (error) {
    console.error("Error verifying proof:", error);
    return NextResponse.json({
      status: "error",
      result: false,
      reason: "Internal Error",
      error_code: "INTERNAL_ERROR"
    }, { status: 200 });
  }
}
```

### Advanced Usage Patterns

#### Context-Aware Verification
Use different verification rules based on user actions:

```typescript
// Frontend sends context via userDefinedData
const userDefinedData = Buffer.from(JSON.stringify({
  action: 'financial_transaction',
  amount: 50000,
  currency: 'USD',
  timestamp: Date.now()
})).toString('hex').padEnd(128, '0');

// Backend uses context to determine requirements
const configStorage = new InMemoryConfigStore(
  async (userIdentifier: string, userDefinedData: string) => {
    const context = JSON.parse(userDefinedData);
    
    if (context.action === 'financial_transaction') {
      return context.amount > 10000 ? 'high_value_kyc' : 'standard_kyc';
    }
    
    return 'default_verification';
  }
);
```

#### Nullifier Management
Prevent proof reuse by tracking nullifiers:

```typescript
// In-memory storage for demo (use database in production)
const usedNullifiers = new Set<string>();

export async function POST(request: NextRequest) {
  // ... verification logic ...
  
  const result = await verifier.verify(/* ... */);
  
  // Check if nullifier has been used before
  if (usedNullifiers.has(result.discloseOutput.nullifier)) {
    return NextResponse.json(
      { 
        status: 'error',
        result: false,
        reason: 'Proof has already been used',
        error_code: "ALREADY_REGISTERED"
      },
      { status: 200 }
    );
  }
  
  // Store nullifier to prevent reuse
  usedNullifiers.add(result.discloseOutput.nullifier);
  
  // ... rest of response logic ...
}
```

## Types Reference

### VerificationConfig
Configuration object for verification requirements:

```typescript
{
  minimumAge: number;             // Minimum age requirement
  excludedCountries: string[];    // ISO 3-letter country codes to exclude
  ofac: boolean;                  // Enable OFAC sanctions checking
}
```

### VcAndDiscloseProof
Zero-knowledge proof structure:

```typescript
{
  a: [BigNumberish, BigNumberish];
  b: [[BigNumberish, BigNumberish], [BigNumberish, BigNumberish]];
  c: [BigNumberish, BigNumberish];
}
```

### AttestationId
Document type identifiers (numeric values):
- `1` - Electronic passport
- `2` - EU ID card

## Best Practices

### 1. Configuration Management
- **Use appropriate storage**: `DefaultConfigStore` for simple apps, `InMemoryConfigStore` for dynamic requirements
- **Validate configurations**: Ensure your verification requirements make sense for your use case
- **Handle edge cases**: Account for missing configurations and invalid user contexts

### 2. Error Handling
- **Always catch ConfigMismatchError**: This provides detailed information about why verification failed
- **Log errors appropriately**: Help with debugging without exposing sensitive information
- **Provide meaningful error messages**: Help users understand what went wrong

### 3. Security Considerations
- **Store nullifiers**: Prevent proof reuse by tracking used nullifiers
- **Validate input**: Always validate all parameters before calling verify()
- **Use HTTPS**: Ensure your endpoint is secure and accessible
- **Rate limiting**: Implement rate limiting to prevent abuse

### 4. Performance Optimization
- **Reuse verifier instances**: Create the verifier once and reuse it
- **Cache configurations**: Cache frequently used verification configurations
- **Optimize blockchain calls**: The verifier makes blockchain calls to validate merkle roots

### 5. Development and Testing
- **Use mockPassport: true** for development and testing
- **Test different scenarios**: Test with various age requirements, countries, and document types
- **Validate scope matching**: Ensure frontend and backend scopes match exactly
- **Use numeric AttestationIds**: Use `1` for electronic passports, `2` for EU ID cards

## Migration Guide

If you're migrating from an older version that used direct configuration methods:

### Old API (No longer supported):
```typescript
// ❌ These methods no longer exist
verifier.setMinimumAge(18);
verifier.excludeCountries('Iran', 'North Korea');
verifier.enableNameAndDobOfacCheck();
```

### New API:
```typescript
// ✅ Configuration via IConfigStorage
const configStorage = new DefaultConfigStore({
  minimumAge: 18,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
});

const verifier = new SelfBackendVerifier(
  scope,
  endpoint,
  mockPassport,
  allowedIds,
  configStorage,  // Configuration is now passed to constructor
  userIdentifierType
);
```

## Network Information

The verifier automatically connects to the appropriate blockchain network:

- **Mainnet** (Real documents): Celo Mainnet - `https://forno.celo.org`
- **Testnet** (Mock documents): Celo Alfajores - `https://alfajores-forno.celo-testnet.org`

The network selection is automatic based on the `mockPassport` parameter in the constructor.