---
icon: arrow-up-right-from-square
---

# V1 to V2 Migration Guide

This guide helps you migrate from Self Protocol V1 to V2. V2 introduces multi-document support, dynamic configuration, and improved data structures.

## Overview of Changes

### What's New in V2
- **Multi-document support**: E-Passports and EU ID Cards
- **Dynamic configuration**: Switch configurations without redeployment
- **Enhanced data formats**: Pre-extracted, human-readable outputs
- **User context data**: Pass custom data through verification flow
- **Improved error handling**: Detailed configuration mismatch reporting

### Breaking Changes
- Backend SDK constructor requires new parameters
- Configuration methods replaced with interface
- Verification method signature changed
- Frontend requires disclosures object
- Smart contract interfaces updated

## Backend Migration

### 1. Update Dependencies

```bash
npm install @selfxyz/core@latest
```

### 2. Update Constructor

**V1 (Old):**
```javascript
const verifier = new SelfBackendVerifier(
  "my-app-scope",
  "https://api.example.com/verify",
  false  // mock mode
);
```

**V2 (New):**
```javascript
import { SelfBackendVerifier, AttestationId, UserIdType, IConfigStorage } from '@selfxyz/core';

// Define allowed document types
const allowedIds = new Map();
allowedIds.set(AttestationId.E_PASSPORT, true);  // Accept passports
allowedIds.set(AttestationId.EU_ID_CARD, true);  // Accept EU ID cards

// Implement configuration storage
class ConfigStorage implements IConfigStorage {
  async getConfig(configId: string) {
    // Return your verification requirements
    return {
      olderThan: 18,
      excludedCountries: ['IRN', 'PRK'],
      ofac: true
    };
  }
  
  async getActionId(userIdentifier: string, userDefinedData?: string) {
    // Return config ID based on your logic
    return 'default_config';
  }
}

const verifier = new SelfBackendVerifier(
  "my-app-scope",
  "https://api.example.com/verify",
  false,                        // mock mode
  allowedIds,                   // NEW: allowed document types
  new ConfigStorage(),          // NEW: config storage
  UserIdType.UUID               // NEW: user ID type
);
```

### 3. Update Configuration

**V1 (Old):**
```javascript
// Direct method calls
verifier.setMinimumAge(18);
verifier.excludeCountries('Iran', 'North Korea');
verifier.enablePassportNoOfacCheck();
verifier.enableNameOfacCheck();
verifier.enableDobOfacCheck();
```

**V2 (New):**
```javascript
// Configuration via IConfigStorage implementation
class ConfigStorage implements IConfigStorage {
  async getConfig(configId: string) {
    // All configuration in one place
    return {
      olderThan: 18,
      excludedCountries: ['IRN', 'PRK'],  // Use ISO 3-letter codes
      ofac: true  // Single boolean for all OFAC checks
    };
  }
}
```

### 4. Update Verification Method

**V1 (Old):**
```javascript
app.post('/api/verify', async (req, res) => {
  const { proof, publicSignals } = req.body;
  
  try {
    const isValid = await verifier.verify(proof, publicSignals);
    res.json({ valid: isValid });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

**V2 (New):**
```javascript
app.post('/api/verify', async (req, res) => {
  const { attestationId, proof, pubSignals, userContextData } = req.body;
  
  try {
    const result = await verifier.verify(
      attestationId,      // NEW: 1 for passport, 2 for EU ID
      proof,
      pubSignals,
      userContextData     // NEW: hex-encoded context data
    );
    
    if (result.isValidDetails.isValid) {
      res.json({
        status: 'success',
        result: true,
        credentialSubject: result.discloseOutput,
        documentType: attestationId === 1 ? 'passport' : 'eu_id_card'
      });
    } else {
      res.status(400).json({
        status: 'error',
        result: false,
        details: result.isValidDetails
      });
    }
  } catch (error) {
    if (error.name === 'ConfigMismatchError') {
      res.status(400).json({
        status: 'error',
        message: 'Configuration mismatch',
        issues: error.issues
      });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});
```

### 5. Handle New Response Format

**V1 Response:**
```javascript
// Simple boolean
true/false
```

**V2 Response:**
```javascript
{
  attestationId: 1,              // Document type
  isValidDetails: {
    isValid: boolean,            // Overall result
    isOlderThanValid: boolean,   // Age check result
    isOfacValid: boolean         // OFAC check result
  },
  forbiddenCountriesList: [],    // Excluded countries
  discloseOutput: {              // Pre-extracted data
    nationality: "USA",
    olderThan: "21",
    name: ["JOHN", "DOE"],
    dateOfBirth: "01-01-1990",
    issuingState: "USA",
    idNumber: "123456789",
    gender: "M",
    expiryDate: "01-01-2030",
    ofac: [true, true, true]
  },
  userData: {
    userIdentifier: "uuid-here",
    userDefinedData: "custom-data"
  }
}
```

## Frontend Migration

### 1. Update Dependencies

```bash
npm install @selfxyz/qrcode@latest
```

### 2. Update QR Code Configuration

**V1 (Old):**
```javascript
import { SelfAppBuilder } from '@selfxyz/qrcode';

const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app-scope",
  endpoint: "https://api.example.com/verify",
  userId: userId,
  logoBase64: logo
}).build();
```

**V2 (New):**
```javascript
import { SelfAppBuilder } from '@selfxyz/qrcode';

const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app-scope",
  endpoint: "https://api.example.com/verify",
  userId: userId,
  logoBase64: logo,
  version: 2,                          // NEW: Specify V2
  userDefinedData: "custom-data",      // NEW: Optional custom data
  disclosures: {                       // NEW: Must match backend config
    // Verification rules
    minimumAge: 18,
    excludedCountries: ['IRN', 'PRK'],
    ofac: true,
    // Data fields to reveal
    name: true,
    nationality: true,
    dateOfBirth: true
  }
}).build();
```

### 3. Important: Disclosures Object

The `disclosures` object in V2 contains **both** verification rules and data fields:

```javascript
disclosures: {
  // Verification rules (must match backend exactly)
  minimumAge: 18,                      // Age requirement
  excludedCountries: ['IRN', 'PRK'],   // ISO 3-letter codes
  ofac: true,                          // OFAC checking
  
  // Data fields to reveal
  name: true,                          // Full name
  nationality: true,                   // Nationality
  dateOfBirth: true,                   // Date of birth
  issuingState: true,                  // Issuing country
  passportNumber: true,                // Document number
  gender: true,                        // Gender
  expiryDate: true                     // Expiration date
}
```

## Smart Contract Migration

### 1. Update Contract Inheritance

**V1 (Old):**
```solidity
import { IIdentityVerificationHub } from "@selfxyz/contracts/interfaces/IIdentityVerificationHub.sol";

contract MyContract {
    IIdentityVerificationHub public hub;
    
    function verify(/* params */) external {
        // Direct hub integration
    }
}
```

**V2 (New):**
```solidity
import { SelfVerificationRoot } from "@selfxyz/contracts/abstract/SelfVerificationRoot.sol";

contract MyContract is SelfVerificationRoot {
    constructor(
        address _hub,
        bytes32 _scope
    ) SelfVerificationRoot(_hub, _scope) {}
    
    // Override to handle verification results
    function customVerificationHook(
        GenericDiscloseOutputV2 memory output,
        bytes memory userData
    ) internal override {
        // Your business logic here
    }
    
    // Override to provide configuration ID
    function getConfigId(
        bytes32 destinationChainId,
        bytes32 userIdentifier,
        bytes memory userDefinedData
    ) public view override returns (bytes32) {
        return keccak256("my-config");
    }
}
```

### 2. Update Hub Addresses

**V2 Hub Addresses:**
```solidity
// Celo Mainnet
address constant HUB_V2 = 0xe57F4773bd9c9d8b6Cd70431117d353298B9f5BF;

// Celo Testnet (Alfajores)
address constant HUB_V2_STAGING = 0x68c931C9a534D37aa78094877F46fE46a49F1A51;
```

### 3. Handle New Data Structure

**V1 Structure:**
```solidity
struct VcAndDiscloseVerificationResult {
    uint256 attestationId;
    uint256 scope;
    uint256 userIdentifier;
    uint256 nullifier;
    uint256[3] revealedDataPacked;  // Packed data
}
```

**V2 Structure:**
```solidity
struct GenericDiscloseOutputV2 {
    bytes32 attestationId;           // Now bytes32
    uint256 userIdentifier;
    uint256 nullifier;
    string issuingState;             // Pre-extracted
    string[] name;                   // Pre-extracted array
    string idNumber;                 // Renamed from passportNumber
    string nationality;              // Pre-extracted
    string dateOfBirth;             // Pre-extracted format
    string gender;                   // Pre-extracted
    string expiryDate;              // Pre-extracted
    uint256 olderThan;
    bool[3] ofac;                   // Bool array
    uint256[4] forbiddenCountriesListPacked;
}
```

## Common Migration Issues

### 1. Configuration Mismatch

**Problem:** Frontend disclosures don't match backend configuration
```
ConfigMismatchError: Configuration mismatch
```

**Solution:** Ensure frontend and backend have identical settings:
```javascript
// Frontend
disclosures: {
  minimumAge: 18,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
}

// Backend (in getConfig)
return {
  olderThan: 18,  // Note: backend uses 'olderThan'
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
};
```

### 2. Missing Attestation ID

**Problem:** Verification fails with missing attestation ID

**Solution:** Frontend must send attestation ID:
```javascript
const requestBody = {
  attestationId: 1,  // 1 for passport, 2 for EU ID
  proof: proof,
  pubSignals: pubSignals,
  userContextData: userContextData
};
```

### 3. Invalid User Context Data

**Problem:** User context data validation fails

**Solution:** Ensure proper hex encoding:
```javascript
// Create user context data (256 bytes total)
const userContextData = '0x' + '0'.repeat(512);  // 512 hex chars = 256 bytes
```

### 4. Document Type Not Allowed

**Problem:** "Attestation ID is not allowed" error

**Solution:** Add document type to allowedIds:
```javascript
const allowedIds = new Map();
allowedIds.set(AttestationId.E_PASSPORT, true);  // Add passport
allowedIds.set(AttestationId.EU_ID_CARD, true);  // Add EU ID card
```

## Testing Your Migration

### 1. Test with Mock Passports

```javascript
// Use staging/testnet for development
const verifier = new SelfBackendVerifier(
  "test-scope",
  "https://test.ngrok.app/verify",
  true,  // Enable mock mode
  allowedIds,
  configStorage,
  UserIdType.UUID
);

// Disable OFAC for mock passports
class ConfigStorage {
  async getConfig() {
    return {
      olderThan: 18,
      ofac: false  // Must be false for mock passports
    };
  }
}
```

### 2. Test Both Document Types

```javascript
// Test passport verification
const passportResult = await verifier.verify(
  AttestationId.E_PASSPORT,  // 1
  passportProof,
  passportSignals,
  userContextData
);

// Test EU ID card verification
const idCardResult = await verifier.verify(
  AttestationId.EU_ID_CARD,  // 2
  idCardProof,
  idCardSignals,
  userContextData
);
```

### 3. Verify Configuration Switching

```javascript
class DynamicConfigStorage {
  async getConfig(configId: string) {
    switch(configId) {
      case 'strict':
        return { olderThan: 21, ofac: true };
      case 'relaxed':
        return { olderThan: 18, ofac: false };
      default:
        return { olderThan: 18, ofac: true };
    }
  }
  
  async getActionId(userIdentifier: string, userDefinedData?: string) {
    // Select config based on user data
    return userDefinedData === 'premium' ? 'strict' : 'relaxed';
  }
}
```

## Best Practices

### 1. Configuration Management

- Store configurations in a database for easy updates
- Version your configurations for rollback capability
- Use meaningful config IDs (not just hashes)
- Document configuration requirements

### 2. Error Handling

```javascript
try {
  const result = await verifier.verify(/* params */);
} catch (error) {
  if (error.name === 'ConfigMismatchError') {
    // Log detailed issues for debugging
    console.error('Config issues:', error.issues);
    // Return user-friendly message
    return { error: 'Verification configuration error' };
  }
  // Handle other errors
}
```

### 3. Security Considerations

- Always validate attestation IDs
- Store and check nullifiers to prevent replay
- Use appropriate scopes for different use cases
- Never expose configuration details to frontend

### 4. Performance Optimization

- Cache configuration objects
- Reuse verifier instances
- Batch verification requests when possible
- Use connection pooling for RPC calls

## Resources

- [Quickstart Guide](quickstart.md) - Basic V2 setup
- [Basic Integration](../contract-integration/basic-integration.md) - Contract examples
- [Workshop Example](../contract-integration/workshop-example.md) - Simple implementation
- [SDK Reference](../sdk-reference/selfbackendverifier.md) - Detailed API docs

## Need Help?

If you encounter issues during migration:
1. Check the [troubleshooting guide](quickstart.md#common-issues)
2. Review example implementations
3. Report issues at [GitHub Issues](https://github.com/selfxyz/self/issues)