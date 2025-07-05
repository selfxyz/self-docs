# SelfAppBuilder

A React component for generating QR codes for Self identity document verification.

## How Self Identity Verification Works

Self uses zero-knowledge proofs to verify identity document information without exposing the actual document data. Here's the complete flow:

### 1. Frontend Setup (Your Web App)
- **SelfAppBuilder** creates a configuration object defining what you want to verify
- **SelfQRcodeWrapper** displays a QR code that users scan with the Self mobile app
- The QR code contains your verification requirements and a unique session ID

### 2. Mobile App Processing
- User scans QR code with Self mobile app
- App reads the identity document's NFC chip and generates a zero-knowledge proof
- Proof validates requirements (age, nationality, etc.) without revealing actual data
- Mobile app sends proof to your backend via the Self relayer service

### 3. Backend Verification (Your API)
- **SelfBackendVerifier** validates the zero-knowledge proof
- Checks proof against on-chain merkle roots (document validity)
- Verifies all requirements match your configuration
- Returns verification result with disclosed information

## V2 Updates

> **New in V2**: Enhanced flexibility with user-defined data for context-aware verification:
>
> * `version: 2` - Specifies the protocol version
> * `userDefinedData` - Custom data passed to your backend for context-aware verification
> * More granular control over verification requirements

## Constructor

```typescript
new SelfAppBuilder(config: Partial<SelfApp>)
```

The `SelfAppBuilder` configures your application's verification requirements. Think of it as creating a "verification recipe" that tells the mobile app exactly what to prove.

### Parameters

| Parameter         | Type                                                           | Required | Description                                                           |
| ----------------- | -------------------------------------------------------------- | -------- | --------------------------------------------------------------------- |
| `appName`         | string                                                         | Yes      | The name of your application (shown in Self mobile app)              |
| `scope`           | string                                                         | Yes      | A unique identifier for your application (max 25 chars)               |
| `endpoint`        | string                                                         | Yes      | The endpoint that will verify the proof                               |
| `endpointType`    | `'celo'` \| `'https'` \| `'staging_celo'` \| `'staging_https'` | No       | Whether the endpoint verifies proofs on-chain or off-chain            |
| `logoBase64`      | string                                                         | No       | Base64-encoded logo or PNG URL to display in the Self app             |
| `userId`          | string                                                         | Yes      | Unique identifier for the user                                        |
| `userIdType`      | `'uuid'` \| `'hex'`                                            | No       | `'hex'` for on-chain addresses, `'uuid'` for off-chain identification |
| `version`         | number                                                         | Yes (V2) | Protocol version (use `2` for latest)                                 |
| `userDefinedData` | string                                                         | Yes (V2) | 64-byte hex string for custom data                                    |
| `disclosures`     | object                                                         | No       | Disclosure and verification requirements                              |
| `devMode`         | boolean                                                        | No       | Enable development mode for testing                                   |

### Verification Requirements (disclosures)

The `disclosures` object defines what information to request and verify:

| Option              | Type      | Description                                    | Use Case                                     |
| ------------------- | --------- | ---------------------------------------------- | -------------------------------------------- |
| `issuing_state`     | boolean   | Request disclosure of document issuing state   | Country-specific services                    |
| `name`              | boolean   | Request disclosure of the user's name          | KYC, personalized services                   |
| `nationality`       | boolean   | Request disclosure of nationality              | Compliance, eligibility checks               |
| `date_of_birth`     | boolean   | Request disclosure of birth date               | Age verification, compliance                 |
| `passport_number`   | boolean   | Request disclosure of document number          | Unique identification                        |
| `gender`            | boolean   | Request disclosure of gender                   | Demographic analysis                         |
| `expiry_date`       | boolean   | Request disclosure of document expiry date     | Document validity                            |
| `minimumAge`        | number    | Verify the user is at least this age           | Age-gated services (18+, 21+, etc.)         |
| `excludedCountries` | string\[] | Array of ISO 3-letter country codes to exclude | Sanctions compliance, regional restrictions  |
| `ofac`              | boolean   | Enable OFAC compliance check                   | Financial services, sanctions screening      |

### Key Concepts Explained

#### Scope - Your Application's Fingerprint
The `scope` acts as your application's unique fingerprint in the Self ecosystem:

- **Maximum 25 characters** to fit in zero-knowledge circuits
- **Used to prevent replay attacks** - proofs generated for one app can't be used in another

```typescript
// Good scopes
scope: "my-app-prod"
scope: "my-kyc-v2"
scope: "my-defi-verifier"

// Bad scopes (too long)
scope: "my-very-long-application-name-that-exceeds-limit"
```

#### User ID - Tying Proofs to Users
The `userId` links the verification to a specific user in your system:
- **UUID format** (`'uuid'`) for traditional web applications
- **Hex format** (`'hex'`) for blockchain applications (wallet addresses)
- **Must be unique** per user to prevent identity confusion
- **Embedded in the proof** so the backend can verify the intended recipient

```typescript
// Web application
userId: "550e8400-e29b-41d4-a716-446655440000"
userIdType: "uuid"

// Blockchain application
userId: "0x742d35Cc6634C0532925a3b8D238D3f9b8b3E4E3"
userIdType: "hex"
```

#### User Defined Data - Context-Aware Verification
V2 introduces `userDefinedData` for dynamic verification requirements:
- **64 bytes (128 hex characters)** of custom data
- **Passed to your backend** via `IConfigStorage.getActionId()`
- **Enables different verification rules** based on context

```typescript
// Different verification rules based on transaction amount
const userDefinedData = JSON.stringify({
  action: "transfer",
  amount: 50000,
  currency: "USD"
});

// Convert to hex and pad to 64 bytes
const hexData = Buffer.from(userDefinedData).toString('hex').padEnd(128, '0');
```

#### Endpoint Requirements
Your verification endpoint must be:
- **Publicly accessible** (not localhost) for production
- **HTTPS enabled** for security
- **Reachable by Self's relayer service** to deliver proofs
- **Implemented with SelfBackendVerifier** for proper validation

For development, use tunneling services like ngrok:
```bash
# Development setup
ngrok http 3000
# Use the ngrok URL as your endpoint
endpoint: "https://abc123.ngrok.io/api/verify"
```

## Complete Integration Example

Here's how the pieces fit together in a real application:

### Frontend (React Component)
```typescript
"use client";

import React, { useState, useEffect, useMemo } from "react";
import { useRouter } from "next/navigation";
import { countries, getUniversalLink } from "@selfxyz/core";
import {
  SelfQRcodeWrapper,
  SelfAppBuilder,
  type SelfApp,
} from "@selfxyz/qrcode";
import { ethers } from "ethers";

export default function Home() {
  const router = useRouter();
  const [selfApp, setSelfApp] = useState<SelfApp | null>(null);
  const [universalLink, setUniversalLink] = useState("");
  const [userId, setUserId] = useState(ethers.ZeroAddress);
  const excludedCountries = useMemo(() => [countries.NORTH_KOREA], []);

  useEffect(() => {
    try {
      const app = new SelfAppBuilder({
        version: 2,
        appName: process.env.NEXT_PUBLIC_SELF_APP_NAME || "Self Workshop",
        scope: process.env.NEXT_PUBLIC_SELF_SCOPE || "self-workshop",
        endpoint: `${process.env.NEXT_PUBLIC_SELF_ENDPOINT}`,
        logoBase64: "https://i.postimg.cc/mrmVf9hm/self.png",
        userId: userId,
        endpointType: "staging_https",
        userIdType: "hex",
        userDefinedData: "Bonjour Cannes!",
        disclosures: {
          minimumAge: 18,
          nationality: true,
          gender: true,
        }
      }).build();

      setSelfApp(app);
      setUniversalLink(getUniversalLink(app));
    } catch (error) {
      console.error("Failed to initialize Self app:", error);
    }
  }, []);

  const handleSuccessfulVerification = () => {
    router.push("/verified");
  };

  return (
    <div className="min-h-screen w-full bg-gray-50 flex flex-col items-center justify-center p-4">
      <div className="bg-white rounded-xl shadow-lg p-6 w-full max-w-md mx-auto">
        <div className="flex justify-center mb-6">
          {selfApp ? (
            <SelfQRcodeWrapper
              selfApp={selfApp}
              onSuccess={handleSuccessfulVerification}
              onError={() => {
                console.error("Error: Failed to verify identity");
              }}
            />
          ) : (
            <div className="w-[256px] h-[256px] bg-gray-200 animate-pulse flex items-center justify-center">
              <p className="text-gray-500 text-sm">Loading QR Code...</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

### Backend (API Endpoint)
```typescript
import { SelfBackendVerifier, DefaultConfigStore } from '@selfxyz/core';

const configStore = new DefaultConfigStore({
  minimumAge: 21,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
});

const verifier = new SelfBackendVerifier(
  'myservice-prod',  // Same scope as frontend
  'https://api.myservice.com/verify',
  false,  // Production mode
  AllIds,  // Accept all document types
  configStore,
  'uuid'  // User ID type
);

export async function POST(request: Request) {
  const { attestationId, proof, pubSignals, userContextData } = await request.json();
  
  try {
    const result = await verifier.verify(
      attestationId,
      proof,
      pubSignals,
      userContextData
    );
    
    if (result.isValidDetails.isValid && result.isValidDetails.isOlderThanValid) {
      return Response.json({ 
        verified: true,
        age_verified: true,
        nationality: result.discloseOutput.nationality
      });
    }
    
    return Response.json({ verified: false }, { status: 400 });
  } catch (error) {
    return Response.json({ error: error.message }, { status: 500 });
  }
}
```

## Migration from V1

V2 introduces `userDefinedData` for more flexible verification scenarios:

```javascript
// V1 - Static configuration
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app",
  endpoint: "https://api.myapp.com/verify",
  userId: userId,
  disclosures: { minimumAge: 18 }
}).build();

// V2 - Dynamic configuration based on context
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app",
  endpoint: "https://api.myapp.com/verify",
  userId: userId,
  version: 2,
  userDefinedData: Buffer.from(JSON.stringify({
    action: "premium_signup",
    tier: "gold"
  })).toString('hex').padEnd(128, '0'),
  disclosures: { minimumAge: 18 }
}).build();
```

## Best Practices

1. **Use Descriptive App Names**: Users see this in the Self mobile app
2. **Keep Scopes Short**: Stay under 25 characters for compatibility
3. **Validate User Input**: Always validate `userId` format matches `userIdType`
4. **Handle Errors Gracefully**: Implement proper error handling in `onSuccess`/`onError`
5. **Test with Dev Mode**: Use `devMode: true` for development and testing
6. **Minimize Data Requests**: Only request disclosure of data you actually need
7. **Use Appropriate Age Checks**: Set `minimumAge` based on your legal requirements
8. **Consider Regional Restrictions**: Use `excludedCountries` for compliance

## Common Use Cases

### Basic Age Verification (18+)
```typescript
disclosures: {
  minimumAge: 18
}
```

### KYC with Name and Nationality
```typescript
disclosures: {
  name: true,
  nationality: true,
  minimumAge: 18,
  ofac: true
}
```

### Geographic Restrictions
```typescript
disclosures: {
  nationality: true,
  excludedCountries: ['IRN', 'PRK', 'CUB']  // Sanctions compliance
}
```