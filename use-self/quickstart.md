---
icon: bullseye-arrow
---

# Quickstart

> ⚠️ **Important**: The Self SDK is currently undergoing significant changes. This guide reflects the latest implementation. If you encounter issues, please check the [SDK repository](https://github.com/selfxyz/self) for updates.

To use Self in your web app, you will display QR codes to request proofs from your front-end, then verify them in your back-end or onchain. This means you will integrate two SDKs:

* The front-end SDK generates and displays QR codes containing information from your app and what you want users to disclose.
* The back-end SDK verifies proofs on a node server (as in this quickstart) or [directly onchain](../contract-integration/basic-integration.md).

## Add `SelfBackendVerifier` to your back-end

### Requirements

* Node v16+

### Install dependencies

{% tabs %}
{% tab title="npm" %}
```bash
npm install @selfxyz/core 
```
{% endtab %}

{% tab title="yarn" %}
```bash
yarn add @selfxyz/core 
```
{% endtab %}

{% tab title="bun" %}
```bash
bun install @selfxyz/core 
```
{% endtab %}
{% endtabs %}

### Set Up SelfBackendVerifier

> ⚠️ **Breaking Change**: The SelfBackendVerifier constructor has changed significantly. It now requires implementing a configuration storage interface and specifying allowed attestation types.

**What changed:**

* ❌ Removed: RPC URL parameter (now handled internally)
* ✅ Added: Configuration storage interface (`IConfigStorage`)
* ✅ Added: Allowed attestation IDs (passport, EU ID card)
* ✅ Added: User identifier type specification (`UserIdType.UUID` or `UserIdType.CUSTOM`)

First, implement the configuration storage interface:

```javascript
import { IConfigStorage, VerificationConfig } from '@selfxyz/core';

class SimpleConfigStorage {
  async getConfig(configId) {
    // Return your verification requirements
    return {
      olderThan: 18,                    // Minimum age (optional)
      excludedCountries: ['IRN', 'PRK'], // ISO 3-letter codes (optional)
      ofac: true                        // Enable OFAC checks (optional)
    };
  }
  
  async getActionId(userIdentifier, userDefinedData) {
    // Return a config ID based on your business logic
    return 'default_config';
  }
}

```

Then set up the verifier:

```javascript
import { SelfBackendVerifier, AttestationId, UserIdType } from '@selfxyz/core';
const IdType = {
  Passport: 1,
  EU_ID_Card: 2,
  // Add other ID types as needed
};
// Define which attestation types to accept
const allowedIds = new Map();
allowedIds.set(IdType.Passport, true); // 1 = passport
allowedIds.set(IdType.EU_ID_Card, true); // 2 = EU ID card (optional)

// Create configuration storage
const configStorage = new SimpleConfigStorage();

// Initialize the verifier
const selfBackendVerifier = new SelfBackendVerifier(
  "my-app-scope",                    // Your app's unique scope
  "https://myapp.com/api/verify",    // The API endpoint of this backend
  false,                             // false = real passports, true = mock for testing
  allowedIds,                        // Allowed document types
  configStorage,                     // Configuration storage implementation
  UserIdType.UUID                    // UUID for off-chain, HEX for on-chain addresses
);
```

To set up the SelfBackendVerifier, pass your application scope, endpoint URL, and optionally the user identifier type and passport mode:

{% hint style="warning" %}
Note that the endpoint of the server you're currently working on. This is NOT localhost. You must either set up a DNS and pass that or if you're developing locally, you must tunnel the localhost endpoint using ngrok.
{% endhint %}

```typescript
import { SelfBackendVerifier } from '@selfxyz/core';

// For production with real passports
const selfBackendVerifier = new SelfBackendVerifier(
    "my-app-scope",              // Your app's unique scope
    "https://myapp.com/api/verify" // The API endpoint of this backend
);

// For development/staging with mock passports
const selfBackendVerifier = new SelfBackendVerifier(
    "my-app-scope",              
    "https://myapp-staging.com/api/verify",
    "uuid",                      // User identifier type: "uuid" (default) or "hex" for addresses
    true                         // Use mock passports for testing
);
```

#### Configuration

> ⚠️ **Breaking Change**: Configuration methods like `setMinimumAge()`, `excludeCountries()`, etc. have been removed. All configuration is now handled through the `IConfigStorage` interface.

**What changed:**

* ❌ Removed: Direct configuration methods (`setMinimumAge`, `excludeCountries`, etc.)
* ✅ Added: Configuration through `IConfigStorage.getConfig()`

Configuration is now handled in your `IConfigStorage` implementation:

```javascript
async getConfig(configId) {
  return {
    olderThan: 18,                        // Instead of setMinimumAge(18)
    excludedCountries: ['IRN', 'PRK'],    // Instead of excludeCountries('Iran', 'North Korea')
    ofac: true                            // Instead of enablePassportNoOfacCheck(), etc.
  };
}
```

### Verification

> ⚠️ **Breaking Change**: The verify method now requires additional parameters including attestation ID and user context data.

**What changed:**

* ❌ Old: `verify(proof, publicSignals)`
* ✅ New: `verify(attestationId, proof, pubSignals, userContextData)`

To verify proofs, call the verify method with the new parameters:

```javascript
import { VcAndDiscloseProof, BigNumberish } from '@selfxyz/core';

// The frontend now sends these additional fields
const { attestationId, proof, pubSignals, userContextData } = request.body;

try {
  const result = await selfBackendVerifier.verify(
    attestationId,    // 1 for passport, 2 for EU ID card
    proof,            // The zero-knowledge proof
    pubSignals,       // Public signals array
    userContextData   // Hex string with user context
  );
  
  if (result.isValidDetails.isValid) {
    console.log('Verification successful');
    console.log('User ID:', result.userData.userIdentifier);
  }
} catch (error) {
  if (error.name === 'ConfigMismatchError') {
    console.error('Configuration mismatch:', error.issues);
  }
}
```

This is the format the API returns:

```javascript
{
  attestationId: 1,              // Document type verified
  isValidDetails: {
    isValid: boolean,            // Overall verification status
    isOlderThanValid: boolean,   // Age check result
    isOfacValid: boolean         // OFAC check result
  },
  forbiddenCountriesList: [],    // List of forbidden countries
  discloseOutput: {              // Disclosed passport data
    nationality: string,
    olderThan: string,
    name: string[],
    dateOfBirth: string,
    // ... other fields
  },
  userData: {
    userIdentifier: string,      // User's unique identifier
    userDefinedData: string      // Additional user data
  }
}
```

#### Example API implementation

This is how an example API implementation would look like this:

```javascript
import { NextApiRequest, NextApiResponse } from 'next';
import { 
  SelfBackendVerifier, 
  AttestationId, 
  UserIdType,
  IConfigStorage,
  ConfigMismatchError 
} from '@selfxyz/core';

// Configuration storage implementation
class ConfigStorage {
  async getConfig(configId) {
    return {
      olderThan: 18,
      excludedCountries: ['IRN', 'PRK'],
      ofac: true
    };
  }
  
  async getActionId(userIdentifier, userDefinedData) {
    return 'default_config';
  }
}

// Initialize verifier once
const allowedIds = new Map();
allowedIds.set(1, true); // Accept passports

const selfBackendVerifier = new SelfBackendVerifier(
  'my-application-scope',
  'https://myapp.com/api/verify',
  false,
  allowedIds,
  new ConfigStorage(),
  UserIdType.UUID
);

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'POST') {
    try {
      const { attestationId, proof, pubSignals, userContextData } = req.body;

      if (!attestationId || !proof || !pubSignals || !userContextData) {
        return res.status(400).json({ message: 'Missing required fields' });
      }

      // Verify the proof
      const result = await selfBackendVerifier.verify(
        attestationId,
        proof,
        pubSignals,
        userContextData
      );
      
      if (result.isValidDetails.isValid) {
        // Return successful verification response
        return res.status(200).json({
          status: 'success',
          result: true,
          credentialSubject: result.discloseOutput
        });
      } else {
        // Return failed verification response
        return res.status(400).json({
          status: 'error',
          result: false,
          message: 'Verification failed',
          details: result.isValidDetails
        });
      }
    } catch (error) {
      if (error instanceof ConfigMismatchError) {
        return res.status(400).json({
          status: 'error',
          result: false,
          message: 'Configuration mismatch',
          issues: error.issues
        });
      }
      
      console.error('Error verifying proof:', error);
      return res.status(500).json({
        status: 'error',
        result: false,
        message: error instanceof Error ? error.message : 'Unknown error'
      });
    }
  } else {
    return res.status(405).json({ message: 'Method not allowed' });
  }
}
```

## Add the QR code generator to your front-end

`QRCodeGenerator` is a React component for generating QR codes for Self passport verification.

### Installation

{% tabs %}
{% tab title="npm" %}
```bash
npm install @selfxyz/qrcode
```
{% endtab %}

{% tab title="yarn" %}
```bash
yarn add @selfxyz/qrcode
```
{% endtab %}

{% tab title="bun" %}
```bash
bun install @selfxyz/qrcode
```
{% endtab %}
{% endtabs %}

### Basic Usage

**1. Import the SelfQRcode component**

```javascript
import SelfQRcodeWrapper, { SelfAppBuilder } from '@selfxyz/qrcode';
import { v4 as uuidv4 } from 'uuid';
```

**2. Create a SelfApp instance using SelfAppBuilder**

> ✅ **New Feature**: The `SelfAppBuilder` now supports a `disclosures` object to specify verification requirements that must match your backend configuration.

```javascript
// Generate a unique user ID
const userId = uuidv4();

// Create a SelfApp instance using the builder pattern
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app-scope", 
  endpoint: "https://myapp.com/api/verify",
  logoBase64: "<base64EncodedLogo>", // Optional, accepts also PNG url
  userId,
  disclosures: {                      // NEW: Specify verification requirements
    minimumAge: 18,                   // Must match backend config
    excludedCountries: ['IRN', 'PRK'], // Must match backend config
    ofac: true,                       // Must match backend config
    nationality: true,                // Request nationality disclosure
    name: true,                       // Request name disclosure
    dateOfBirth: true                 // Request date of birth disclosure
  }
}).build();
```

Note that if you're choosing the endpointType to be https , the endpoint field must be accessible for anyone to call it (i.e., not localhost). The reason is that the Self backend relayer calls this endpoint to verify the proof.

Be careful and use the same scope here as you used in the backend code shown above.

> ⚠️ **Important**: The `disclosures` object MUST match your backend's `VerificationConfig`. Any mismatch will cause verification failures.

**3. Render the QR code component**

```javascript
function MyComponent() {
  return (
    <SelfQRcodeWrapper
      selfApp={selfApp}
      onSuccess={() => {
        console.log('Verification successful');
        // Perform actions after successful verification
      }}
    />
  );
}
```

SelfQRcodeWrapper wraps SelfQRcode to prevent server-side rendering when using nextjs. When not using nextjs, SelfQRcode can be used instead.

Your scope is an identifier for your application. It makes sure people can't use proofs destined for other applications in yours. You'll have to use the same scope in the backend verification SDK if you need to verify proofs offchain, or in your contract if you verify them onchain. Make sure it's no longer than 25 characters.

The userId is the unique identifier of your user. It ties them to their proof. If you want you can use a standard uuid. If you want to verify it onchain, you should use the user's address so no one can steal their proof and use it with another address.

To see how you can configure your SelfApp take a look atSelfAppBuilder. You can also find the SDK reference forSelfQRcodeWrapper.

#### &#x20;To see how you can configure your `SelfApp`  take a look at[selfappbuilder.md](../sdk-reference/selfappbuilder.md "mention"). You can also find the SDK reference for[selfqrcodewrapper.md](../sdk-reference/selfqrcodewrapper.md "mention").

### Complete Example

Here's a complete example of how to implement the Self QR code in a NextJS application:

```javascript
'use client';

import React, { useState, useEffect } from 'react';
import SelfQRcodeWrapper, { SelfAppBuilder } from '@selfxyz/qrcode';
import { v4 as uuidv4 } from 'uuid';

function VerificationPage() {
  const [userId, setUserId] = useState<string | null>(null);

  useEffect(() => {
    // Generate a user ID when the component mounts
    setUserId(uuidv4());
  }, []);

  if (!userId) return null;

  // Create the SelfApp configuration
  const selfApp = new SelfAppBuilder({
    appName: "My Application",
    scope: "my-application-scope",
    endpoint: "https://myapp.com/api/verify",
    userId,
    disclosures: {                    // NEW: Must match backend config
      minimumAge: 18,
      excludedCountries: ['IRN', 'PRK'],
      ofac: true,
      name: true,
      nationality: true
    }
  }).build();

  return (
    <div className="verification-container">
      <h1>Verify Your Identity</h1>
      <p>Scan this QR code with the Self app to verify your identity</p>
      
      <SelfQRcodeWrapper
        selfApp={selfApp}
        onSuccess={() => {
          // Handle successful verification
          console.log("Verification successful!");
          // Redirect or update UI
        }}
        size={350}
      />
      
      <p className="text-sm text-gray-500">
        User ID: {userId.substring(0, 8)}...
      </p>
    </div>
  );
}

export default VerificationPage;
```

### Example

For a more comprehensive and interactive example, please refer to the [playground](https://github.com/selfxyz/playground/blob/main/app/page.tsx).

### Verification Flow

```
onSuccess
```

The QR code component displays the current verification status with an LED indicator and changes its appearance based on the verification state.

### What's New and Breaking Changes

#### Backend SDK Changes

1. **Constructor Changes**:
   * Removed RPC URL parameter (now handled internally based on mock mode)
   * Added required `IConfigStorage` implementation
   * Added `allowedIds` Map for attestation types
   * Added `UserIdType` specification
2. **Configuration Changes**:
   * Removed all direct configuration methods (`setMinimumAge`, etc.)
   * Configuration now handled through `IConfigStorage` interface
3. **Verification Changes**:
   * New parameters: `attestationId`, `userContextData`
   * Different return structure with more detailed information

#### Frontend SDK Changes

1. **New `disclosures` Object**:
   * Specify verification requirements in frontend
   * Must match backend configuration exactly

#### Key Concepts to Understand

* **Attestation IDs**: `1` for passport, `2` for EU ID card
* **IConfigStorage**: Interface for managing verification configurations
* **User Context Data**: Hex-encoded string with user information
* **Configuration Matching**: Frontend and backend must have identical verification requirements
