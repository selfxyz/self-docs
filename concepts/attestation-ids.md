# attestation IDs

Attestation IDs are numeric identifiers that specify which type of identity document is being verified in the Self Protocol. They are a core concept in V2 and are used throughout the verification flow.

### Current Attestation Types

| ID | Document Type       | Description                                     |
| -- | ------------------- | ----------------------------------------------- |
| 1  | Electronic Passport | NFC-enabled passports with biometric chip       |
| 2  | EU ID Card          | European Union national identity cards with NFC |

### Usage in Backend

#### Configuring Allowed Attestation Types

```typescript
import { SelfBackendVerifier, AttestationId, AllIds } from '@selfxyz/core';

// Option 1: Accept all supported document types
const verifier = new SelfBackendVerifier(
  scope,
  endpoint,
  false,
  AllIds, // Includes all supported attestation types
  configStorage,
  'uuid'
);

// Option 2: Accept only passports
const allowedIds = new Map<AttestationId, boolean>();
allowedIds.set(1, true); // Only passports

const verifier = new SelfBackendVerifier(
  scope,
  endpoint,
  false,
  allowedIds,
  configStorage,
  'uuid'
);

// Option 3: Accept both passports and EU ID cards
const allowedIds = new Map<AttestationId, boolean>();
allowedIds.set(1, true); // Passports
allowedIds.set(2, true); // EU ID cards
```

#### Verification with Attestation ID

The attestation ID is passed as the first parameter to the verify method:

```typescript
const result = await verifier.verify(
  attestationId,    // 1 for passport, 2 for EU ID
  proof,
  pubSignals,
  userContextData
);

// The result includes which document type was verified
console.log(`Verified document type: ${result.attestationId}`);
```

### Usage in Smart Contracts

In V2 smart contracts, attestation IDs are used to determine verification parameters:

```solidity
// Example from the V2 contract architecture
function getConfigId(
    uint256 destinationChainId,
    address userIdentifier,
    bytes calldata userDefinedData,
    uint256 attestationId  // Document type identifier
) public view returns (bytes32) {
    // Different configurations based on document type
    if (attestationId == 1) {
        // Passport-specific logic
        return keccak256(abi.encodePacked("passport_config"));
    } else if (attestationId == 2) {
        // EU ID card specific logic
        return keccak256(abi.encodePacked("eu_id_config"));
    }
}
```

### Attestation-Specific Features

Different attestation types may have different capabilities:

#### Passport (ID: 1)

* Full biometric data access
* Global coverage
* Longer validity periods
* More extensive OFAC checking capabilities

#### EU ID Card (ID: 2)

* Limited to EU citizens
* May have shorter validity periods
* Different data fields available
* Region-specific compliance requirements

### Handling Multiple Attestation Types

#### Dynamic Configuration Based on Attestation Type

```typescript
class AttestationAwareConfigStore implements IConfigStorage {
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
    
    // Different configs for different document types
    if (data.attestationId === 1) {
      return 'passport_verification';
    } else if (data.attestationId === 2) {
      return 'eu_id_verification';
    }
    
    return 'default_verification';
  }
  
  async getConfig(configId: string): Promise<VerificationConfig> {
    switch (configId) {
      case 'passport_verification':
        return {
          olderThan: 18,
          excludedCountries: ['IRN', 'PRK'],
          ofac: true
        };
      
      case 'eu_id_verification':
        return {
          olderThan: 18,
          excludedCountries: [], // EU IDs might have different restrictions
          ofac: false
        };
      
      default:
        return { olderThan: 18 };
    }
  }
}
```

#### Frontend Considerations

While the frontend doesn't directly specify the attestation ID (the user's document type determines this), you can design your UI to handle different document types:

```javascript
function VerificationComponent({ acceptedDocuments }) {
  const selfApp = new SelfAppBuilder({
    // ... other config
    disclosures: {
      // These apply regardless of document type
      minimumAge: 18,
      nationality: true
    }
  }).build();
  
  return (
    <div>
      <h2>Accepted Documents:</h2>
      <ul>
        {acceptedDocuments.includes(1) && <li>✓ Electronic Passports</li>}
        {acceptedDocuments.includes(2) && <li>✓ EU ID Cards</li>}
      </ul>
      <SelfQRcodeWrapper selfApp={selfApp} onSuccess={handleSuccess} />
    </div>
  );
}
```

### Error Handling

When a user tries to verify with a document type that's not allowed:

```typescript
try {
  const result = await verifier.verify(attestationId, proof, pubSignals, userContextData);
} catch (error) {
  if (error.name === 'ConfigMismatchError') {
    const invalidIdIssue = error.issues.find(
      issue => issue.type === 'InvalidId'
    );
    
    if (invalidIdIssue) {
      console.error(`Document type ${attestationId} is not accepted`);
      // Show appropriate error to user
    }
  }
}
```

### Future Extensibility

The attestation ID system is designed to be extensible. Future document types might include:

* National ID cards from other regions
* Driver's licenses with NFC
* Residence permits
* Other government-issued identity documents

Each new document type would receive a unique attestation ID and could have its own:

* Verification circuits
* Data fields
* Compliance requirements
* Geographic restrictions

### Best Practices

1. **Always validate attestation IDs** in your backend before processing
2. **Configure appropriate restrictions** based on your compliance requirements
3. **Provide clear user feedback** about which document types are accepted
4. **Test with both document types** if you accept multiple
5. **Log attestation types** for audit and analytics purposes
6. **Consider regional requirements** when accepting different document types

### Integration with Verification Flow

The complete flow with attestation IDs:

1. User scans QR code with Self app
2. App detects document type (passport or EU ID)
3. App generates proof with appropriate attestation ID
4. Backend receives attestation ID with proof
5. Backend validates attestation ID is allowed
6. Backend applies document-specific verification logic
7. Result includes which document type was verified

This system ensures flexibility while maintaining security and compliance across different identity document types.
