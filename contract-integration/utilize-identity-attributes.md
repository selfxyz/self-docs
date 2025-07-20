# Utilize Identity Attributes V2

> **V2 Simplified Access**: V2 provides direct access to verified attributes through the `GenericDiscloseOutputV2` structure, eliminating the need for complex data extraction in most cases.

Self Protocol supports multiple identity document types, each providing rich identity attributes. This guide shows how to access and utilize verified identity data from supported documents:

- **E_PASSPORT**: International passports with machine-readable zones
- **EU_ID_CARD**: European Union national identity cards

All document types provide standardized access to core identity attributes through a unified interface.

## V2 Direct Access (Recommended)

V2 provides structured identity data directly through `GenericDiscloseOutputV2`. All supported document types use the same unified interface:

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory userData
) internal override {
    // Universal identity attributes (available for all document types)
    string memory userNationality = output.nationality;
    string[] memory userName = output.name; // [first, middle, last]
    string memory documentNumber = output.idNumber; // Document number (passport/ID)
    string memory issuingCountry = output.issuingState;
    string memory birthDate = output.dateOfBirth; // DD-MM-YY format
    string memory gender = output.gender;
    string memory expiryDate = output.expiryDate;
    uint256 verifiedAge = output.olderThan;
    
    // OFAC verification results
    bool documentOfacClear = output.ofac[0]; // Document number OFAC check
    bool nameAndDobOfacClear = output.ofac[1]; // Name + DOB OFAC check
    bool nameAndYobOfacClear = output.ofac[2]; // Name + YOB OFAC check
    
    // Document type-specific handling
    if (output.attestationId == AttestationId.E_PASSPORT) {
        // Passport-specific business logic
        // All three OFAC modes available for passports
        handlePassportVerification(output);
    } else if (output.attestationId == AttestationId.EU_ID_CARD) {
        // EU ID card-specific business logic
        // Only nameAndDobOfac and nameAndYobOfac available (documentOfacClear = false)
        handleEuIdCardVerification(output);
    }
}
```

### Document Type Support

| Feature | E_PASSPORT | EU_ID_CARD |
|---------|------------|------------|
| Identity Attributes | ✅ All available | ✅ All available |
| Document Number OFAC | ✅ Supported | ❌ Not available |
| Name + DOB OFAC | ✅ Supported | ✅ Supported |
| Name + YOB OFAC | ✅ Supported | ✅ Supported |
| Geographic Restrictions | ✅ Supported | ✅ Supported |
| Age Verification | ✅ Supported | ✅ Supported |

## Advanced Usage

### Direct Circuit Data Access

For advanced use cases requiring direct access to circuit outputs, you can extract attributes from the underlying `PassportOutput` or `EuIdOutput` structures:

```solidity
// Extract attributes directly from circuit data
function extractAdvancedAttributes(
    bytes32 attestationId,
    bytes memory revealedDataPacked
) internal pure returns (
    string memory issuingState,
    string[] memory name,
    string memory documentNumber,
    string memory nationality,
    string memory dateOfBirth,
    string memory gender,
    string memory expiryDate
) {
    // Works for both E_PASSPORT and EU_ID_CARD
    issuingState = CircuitAttributeHandlerV2.getIssuingState(attestationId, revealedDataPacked);
    name = CircuitAttributeHandlerV2.getName(attestationId, revealedDataPacked);
    documentNumber = CircuitAttributeHandlerV2.getDocumentNumber(attestationId, revealedDataPacked);
    nationality = CircuitAttributeHandlerV2.getNationality(attestationId, revealedDataPacked);
    dateOfBirth = CircuitAttributeHandlerV2.getDateOfBirth(attestationId, revealedDataPacked);
    gender = CircuitAttributeHandlerV2.getGender(attestationId, revealedDataPacked);
    expiryDate = CircuitAttributeHandlerV2.getExpiryDate(attestationId, revealedDataPacked);
}
```

### OFAC Compliance Checking

```solidity
// Document-specific OFAC verification
function checkOfacCompliance(
    bytes32 attestationId,
    bytes memory revealedDataPacked
) internal pure returns (bool[3] memory ofacResults) {
    if (attestationId == AttestationId.E_PASSPORT) {
        // Passports: All three OFAC modes available
        ofacResults[0] = CircuitAttributeHandlerV2.getDocumentNoOfac(attestationId, revealedDataPacked);
        ofacResults[1] = CircuitAttributeHandlerV2.getNameAndDobOfac(attestationId, revealedDataPacked);
        ofacResults[2] = CircuitAttributeHandlerV2.getNameAndYobOfac(attestationId, revealedDataPacked);
    } else if (attestationId == AttestationId.EU_ID_CARD) {
        // EU ID cards: Only name-based OFAC checks
        ofacResults[0] = false; // Document number OFAC not supported
        ofacResults[1] = CircuitAttributeHandlerV2.getNameAndDobOfac(attestationId, revealedDataPacked);
        ofacResults[2] = CircuitAttributeHandlerV2.getNameAndYobOfac(attestationId, revealedDataPacked);
    }
}
```

## CircuitAttributeHandlerV2 Library Reference

The `CircuitAttributeHandlerV2` library provides functions for extracting and validating identity attributes from circuit data for all supported document types:

### Identity Attribute Extraction

| Function | Description | Support |
|----------|-------------|----------|
| `getIssuingState(attestationId, data)` | Document issuing country/state | All documents |
| `getName(attestationId, data)` | Full name array [first, middle, last] | All documents |
| `getDocumentNumber(attestationId, data)` | Document number (passport/ID) | All documents |
| `getNationality(attestationId, data)` | Holder's nationality | All documents |
| `getDateOfBirth(attestationId, data)` | Birth date (DD-MM-YY format) | All documents |
| `getGender(attestationId, data)` | Gender information | All documents |
| `getExpiryDate(attestationId, data)` | Document expiry date | All documents |

### OFAC Verification Functions

| Function | Description | E_PASSPORT | EU_ID_CARD |
|----------|-------------|------------|------------|
| `getDocumentNoOfac(attestationId, data)` | Document number OFAC check | ✅ | ❌ |
| `getNameAndDobOfac(attestationId, data)` | Name + DOB OFAC check | ✅ | ✅ |
| `getNameAndYobOfac(attestationId, data)` | Name + YOB OFAC check | ✅ | ✅ |
| `compareOfac(attestationId, data, modes...)` | Multi-mode OFAC validation | ✅ | ✅ Partial |

### Validation Functions

- **`compareOlderThan(attestationId, data, age)`** - Validates minimum age requirement
- **`compareOfac(attestationId, data, modes...)`** - Validates OFAC compliance across multiple modes

> **Note**: All functions require `attestationId` parameter to handle document type differences correctly.