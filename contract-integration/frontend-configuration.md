# Frontend SDK Configuration

Configure the frontend SDK for on-chain verification. For basic setup and installation, see the [Quickstart Guide](../use-self/quickstart.md).

## Contract-Specific Configuration

When integrating with contracts, configure your SelfApp with these key parameters:

```javascript
const selfApp = new SelfAppBuilder({
    // Contract integration settings
    endpoint: "YOUR_DEPLOYED_CONTRACT_ADDRESS",  // Your SelfVerificationRoot contract
    endpointType: "staging_celo",               // "staging_celo" or "celo"
    userIdType: "hex",                          // Use "hex" for wallet addresses
    version: 2,                                 // Always use V2 for contracts
    
    // Your app details
    appName: "Your App Name",
    scope: "your-app-scope",                    // Max 30 characters
    userId: userWalletAddress,
    
    // Verification configuration (must match your contract)
    disclosures: { /* see below */ },
    userDefinedData: "",                        // Optional: dynamic data for contract
}).build();
```

## Key Configuration for Contracts

**Contract Integration:**
- `endpoint`: Your deployed contract address
- `endpointType`: `"staging_celo"` (testnet) or `"celo"` (mainnet)  
- `userIdType`: Use `"hex"` for wallet addresses

**Important:** Your `disclosures` configuration must exactly match your contract's verification requirements.

## Disclosures Configuration

Configure what users must verify and what data they reveal:

### Verification Rules
```javascript
disclosures: {
    // Age verification
    minimumAge: 18,                    // Minimum age requirement
    
    // Geographic restrictions  
    excludedCountries: ["USA", "RUS"], // Array of 3-letter country codes
    
    // Compliance checking
    ofac: false,                       // OFAC sanctions list checking
}
```

### Data Disclosures  
```javascript
disclosures: {
    // Personal information
    name: true,                       // Full name
    date_of_birth: true,              // Date of birth
    gender: true,                     // Gender
    
    // Document information
    nationality: true,                // Nationality/citizenship
    issuing_state: true,              // Document issuing country
    passport_number: true,            // Passport number
    expiry_date: true,                // Document expiration date
}
```

> **Important:** Your frontend disclosures must match your contract's verification configuration.

### User Defined Data

The `userDefinedData` parameter allows you to pass arbitrary string data through the verification flow to your contract. This data is cryptographically included in the verification process and cannot be tampered with.

**Common Use Cases:**
- **Dynamic Configuration:** Select different verification configs based on user context
- **Business Logic:** Include application-specific data for custom verification logic

**Data Flow:**
1. Frontend sets `userDefinedData` in SelfAppBuilder
2. Data flows through mobile app and TEE processing
3. Contract receives data in both `getConfigId()` and `customVerificationHook()` functions

**Contract Access:**
```solidity
// In getConfigId - used for configuration selection
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier,
    bytes memory userDefinedData  // ← Your custom data here
) public view override returns (bytes32) {
    // Return your configuration ID
    return YOUR_CONFIG_ID;
}

// In customVerificationHook - userData contains full context including userDefinedData
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory userData  // ← First 64 bytes are destChainId+userIdentifier, rest is userDefinedData
) internal override {
    bytes memory userDefinedData = userData[64:];  // Extract custom data
    // Implement your business logic based on userDefinedData
}
```

## Common Contract Integration Examples

### Age-Gated Contract (21+)
```javascript
disclosures: { 
    minimumAge: 21,
    date_of_birth: true  // Optional: reveal birth date
}
```

### Geographic Restrictions
```javascript  
disclosures: { 
    excludedCountries: ["USA", "RUS"],
    nationality: true,      // Required for geo-filtering
    issuing_state: true     // Optional: additional geo data
}
```

### Dynamic Configuration with userDefinedData
```javascript
// Pass action type for dynamic config selection
const selfApp = new SelfAppBuilder({
    endpoint: "YOUR_CONTRACT_ADDRESS",
    userDefinedData: "0x01",  // Action type for contract routing
    disclosures: { 
        minimumAge: 18,
        nationality: true
    }
    // ... other config
}).build();
```