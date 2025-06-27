# SDK Configuration

Below is an example of how to configure the QR code for on-chain verification using our SDK.

```typescript
const selfApp = new SelfAppBuilder({
        appName: "Self Example App V2",
        scope: "Self-Example-App-V2",
        endpoint: "YOUR_DEPLOYED_CONTRACT_ADDRESS", // Your SelfVerificationRoot contract
        endpointType: "staging_celo", // "staging_celo" for testnet, "celo" for mainnet
        logoBase64: logo,
        userId: address, // User's wallet address (required)
        userIdType: "hex", // "uuid" or "hex"
        version: 2, // V2 configuration
        disclosures: { 
            // Passport data fields
            date_of_birth: true,
            nationality: true,
            name: true,
            issuing_state: true,
            passport_number: true, // Passport number field
            gender: true,
            expiry_date: true,
            
            // Verification rules (integrated in disclosures for V2)
            minimumAge: 18, // Age requirement (10-100)
            excludedCountries: [], // Array of 3-letter country codes (e.g., ["USA", "RUS"])
            ofac: true // OFAC compliance checking (boolean)
        },
        devMode: true, // Set to false for production
}).build();
```

## Configuration Parameters

**Required Parameters:**
- `appName`: Name of your application
- `scope`: Application identifier (max 31 ASCII characters)
- `endpoint`: Your deployed `SelfVerificationRoot` contract address
- `userId`: User's unique identifier (wallet address or UUID)

**Optional Parameters:**
- `endpointType`: `"celo"` (mainnet) or `"staging_celo"` (testnet)
- `userIdType`: `"uuid"` or `"hex"` (default: "uuid")
- `version`: SDK version (default: 2 for V2)
- `logoBase64`: Base64-encoded logo for the Self app
- `devMode`: Development mode flag (default: false)
- `disclosures`: Identity attributes and verification rules

**V2 Disclosures:**
- Passport data fields: `name`, `nationality`, `date_of_birth`, `issuing_state`, `passport_number`, `gender`, `expiry_date`
- Verification rules: `minimumAge`, `excludedCountries`, `ofac`

**Important:** Your verification configuration in the SDK should match the configuration set in your contract.

### Available Disclosures

```typescript
disclosures: {
    // Passport data fields (boolean)
    name?: boolean,                 // Full name
    date_of_birth?: boolean,        // Date of birth
    nationality?: boolean,          // Nationality/citizenship
    gender?: boolean,               // Gender
    issuing_state?: boolean,        // Document issuing country
    passport_number?: boolean,      // Passport number
    expiry_date?: boolean,         // Document expiration date
    
    // Verification rules
    minimumAge?: number,           // Minimum age requirement (10-100)
    excludedCountries?: string[],  // Array of 3-letter country codes (max 40)
    ofac?: boolean,               // OFAC compliance checking
}
```

For detailed passport attribute usage, see [Identity Attributes](utilize-passport-attributes.md).

### Configuration Examples

**Age Verification (21+):**
```typescript
const selfApp = new SelfAppBuilder({
    appName: "Age Verification App",
    scope: "Age-Verification-App",
    endpoint: "YOUR_CONTRACT_ADDRESS",
    userId: userAddress,
    endpointType: "staging_celo",
    disclosures: { 
        date_of_birth: true,
        minimumAge: 21
    }
}).build();
```

**Geographic Restrictions:**
```typescript
const selfApp = new SelfAppBuilder({
    appName: "Geographic Restricted App",
    scope: "Geographic-App",
    endpoint: "YOUR_CONTRACT_ADDRESS", 
    userId: userAddress,
    disclosures: { 
        nationality: true, 
        issuing_state: true,
        excludedCountries: ["USA", "RUS", "CHN"]
    }
}).build();
```

**OFAC Compliance:**
```typescript
const selfApp = new SelfAppBuilder({
    appName: "OFAC Compliant App",
    scope: "OFAC-App",
    endpoint: "YOUR_CONTRACT_ADDRESS",
    userId: userAddress,
    disclosures: { 
        name: true, 
        date_of_birth: true, 
        ofac: true
    }
}).build();
```