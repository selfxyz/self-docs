---
description: Integrate Self protocol inside your mobile application using Deeplinking
icon: mobile
---

# Use deeplinking



> ⚠️ **Breaking Changes**:
>
> * `SelfAppBuilder` is now imported from `@selfxyz/qrcode`, not `@selfxyz/core`
> * New `disclosures` object is required to specify verification requirements
> * Additional required parameters in the builder

Import getUniversalLink from `@selfxyz/core` and SelfAppBuilder from `@selfxyz/qrcode`:

```javascript
import { getUniversalLink } from '@selfxyz/core';
import { SelfAppBuilder } from '@selfxyz/qrcode';
```

Instantiate your Self app using SelfAppBuilder with the required configuration:

```javascript
const selfApp = new SelfAppBuilder({
    appName: "Your App Name",              // Required: Your application name
    scope: "your-app-scope",               // Required: Unique identifier for your app (max 31 chars)
    endpoint: "https://your-api.com/verify", // Required: Your verification endpoint
    logoBase64: "<base64-logo-or-png-url>", // Optional: Your app logo
    userId: "user-uuid-or-address",        // Required: User identifier
    userIdType: 'uuid',                    // Optional: 'uuid' (default) or 'hex' for addresses
    version: 2,                            // NEW in V2: Protocol version
    userDefinedData: "0x" + Buffer.from("default").toString('hex').padEnd(128, '0'), // NEW in V2
    disclosures: {                         // NEW: Specify what to verify
        minimumAge: 18,                    // Age verification
        excludedCountries: ['IRN', 'PRK'], // ISO 3-letter country codes to exclude
        ofac: true,                        // OFAC sanctions check
        name: true,                        // Request name disclosure
        nationality: true,                 // Request nationality disclosure
        date_of_birth: true,               // Request date of birth
        // ... other optional fields
    }
}).build();
```

Get the deeplink from the Self app object:

```javascript
const deeplink = getUniversalLink(selfApp);
```

You can now use this deeplink to redirect your users to the Self mobile app.

### What's New in V2

#### Required Parameters

* `version: 2` - Specifies the protocol version
* `userDefinedData` - 64-byte hex string for custom data passed to your backend

#### Disclosures Object

The `disclosures` object allows you to specify what information you want to verify and request from the user's passport:

```javascript
disclosures: {
    // Identity fields (optional)
    issuing_state?: boolean,      // Country that issued the passport
    name?: boolean,               // Full name
    passport_number?: boolean,    // Passport number
    nationality?: boolean,        // Nationality
    date_of_birth?: boolean,      // Date of birth
    gender?: boolean,             // Gender
    expiry_date?: boolean,        // Passport expiry date
    
    // Verification requirements (optional)
    minimumAge?: number,          // Minimum age requirement (e.g., 18, 21)
    excludedCountries?: string[], // ISO 3-letter codes (e.g., ['IRN', 'PRK'])
    ofac?: boolean,               // Enable OFAC sanctions checking
}
```

#### User ID Types

When using blockchain addresses as user IDs:

```javascript
// For Ethereum/blockchain addresses
const selfApp = new SelfAppBuilder({
    // ... other config
    userId: "0x1234567890abcdef...",  // User's address
    userIdType: 'hex',                 // Specify hex for addresses
}).build();

// For regular UUIDs
const selfApp = new SelfAppBuilder({
    // ... other config
    userId: "550e8400-e29b-41d4-a716-446655440000",  // UUID v4
    userIdType: 'uuid',  // Default, can be omitted
}).build();
```

### Important Notes

1. **Endpoint Requirements**:
   * For `endpointType: 'https'` (default), the endpoint must be publicly accessible
   * The endpoint must start with `https://`
   * Maximum length: 496 characters
2. **Scope Limitations**:
   * Must contain only ASCII characters
   * Maximum length: 31 characters
   * Should be unique to your application
3. **V2 Parameters**:
   * Always include `version: 2` for the latest protocol
   * `userDefinedData` is a 64-byte hex string for passing custom data to your backend
   * This data is used by your `IConfigStorage` implementation to determine verification rules
4. **Backend Coordination**:
   * The `disclosures` object must match your backend verification configuration
   * Any mismatch will cause verification failures

### Migration from V1

If you're upgrading from V1, add these new parameters:

```javascript
// V1 (old)
const selfApp = new SelfAppBuilder({
    appName: "My App",
    scope: "my-app",
    endpoint: "https://api.myapp.com/verify",
    userId: userId
}).build();

// V2 (new)
const selfApp = new SelfAppBuilder({
    appName: "My App",
    scope: "my-app", 
    endpoint: "https://api.myapp.com/verify",
    userId: userId,
    version: 2,                    // Add version
    userDefinedData: "0x00...",    // Add user defined data
    disclosures: { /* ... */ }     // Add disclosures
}).build();
```

