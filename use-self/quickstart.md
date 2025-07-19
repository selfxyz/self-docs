---
icon: bullseye-arrow
---

# Quickstart

> ⚠️ **Important**: The Self SDK is currently undergoing significant changes. This guide reflects the latest implementation. If you encounter issues, please check the [SDK repository](https://github.com/selfxyz/self) for updates.

## Before You Start

**New to Self Protocol?** We highly recommend watching our [ETHGlobal Cannes Workshop](https://www.youtube.com/live/0Jg1o9BFUBs?si=4g0okIn91QMIzew-) first. This essential workshop walks through the core concepts and provides a hands-on introduction to building with Self.

## Overview

To use Self in your web app, you will display QR codes to request proofs from your front-end, then verify them in your back-end or onchain. This means you will integrate two SDKs:

* The front-end SDK generates and displays QR codes containing information from your app and what you want users to disclose.
* The back-end SDK verifies proofs on a node server (as in this quickstart) or [directly onchain](../contract-integration/basic-integration.md).

## Add the QR code generator to your front-end

`QRCodeGenerator` is a React component for generating QR codes for Self passport verification.

### Installation

Install the required frontend packages:

{% tabs %}
{% tab title="npm" %}
```bash
npm install @selfxyz/qrcode @selfxyz/core ethers
```
{% endtab %}

{% tab title="yarn" %}
```bash
yarn add @selfxyz/qrcode @selfxyz/core ethers
```
{% endtab %}

{% tab title="bun" %}
```bash
bun install @selfxyz/qrcode @selfxyz/core ethers
```
{% endtab %}
{% endtabs %}

**Package purposes:**
- `@selfxyz/qrcode`: QR code generation and display components
- `@selfxyz/core`: Core utilities including `getUniversalLink` for deeplinks
- `ethers`: Ethereum utilities for address handling

### Basic Usage

Here's a simplified approach to setting up the QR code:

**1. Import required components**

```javascript
import SelfQRcodeWrapper, { SelfAppBuilder } from '@selfxyz/qrcode';
import { getUniversalLink } from "@selfxyz/core";
import { ethers } from "ethers";
```

**2. Create and configure your SelfApp**

```javascript
// Use zero address for demo purposes
const userId = ethers.ZeroAddress;

// Create the SelfApp
const selfApp = new SelfAppBuilder({
  version: 2,
  appName: "Self Example",
  scope: "your-app-scope",
  endpoint: process.env.NEXT_PUBLIC_SELF_ENDPOINT || "",
  logoBase64: "https://i.postimg.cc/mrmVf9hm/self.png",
  userId: userId,
  endpointType: "staging_https",
  userIdType: "hex",
  userDefinedData: "Hello World!",
  disclosures: {
    // Verification requirements (must match backend)
    minimumAge: 18,
    // ofac: false,
    // excludedCountries: [],

    // Disclosure requests (what users reveal)
    nationality: true,
    gender: true,
    // Other optional fields:
    // name: false,
    // date_of_birth: true,
    // passport_number: false,
    // expiry_date: false,
  }
}).build();
```

> ⚠️ **Important**: Use the same scope and disclosures configuration as your backend to avoid verification failures.

**3. Display the QR code**

```javascript
function VerificationComponent() {
  return (
    <SelfQRcodeWrapper
      selfApp={selfApp}
      onSuccess={() => {
        console.log('Verification successful');
        // Handle successful verification
      }}
      onError={() => {
        console.error('Failed to verify identity');
      }}
    />
  );
}
```

SelfQRcodeWrapper wraps SelfQRcode to prevent server-side rendering when using nextjs. When not using nextjs, SelfQRcode can be used instead.

Your scope is an identifier for your application. It makes sure people can't use proofs destined for other applications in yours. You'll have to use the same scope in the backend verification SDK if you need to verify proofs offchain. Make sure it's no longer than 25 characters.

To see how you can configure your SelfApp take a look at `SelfAppBuilder`. You can also find the SDK reference for `SelfQRcodeWrapper`.

### Complete Example

Here's a complete Next.js component example based on the workshop:

```javascript
'use client';

import React, { useState, useEffect } from 'react';
import { getUniversalLink } from "@selfxyz/core";
import {
  SelfQRcodeWrapper,
  SelfAppBuilder,
  type SelfApp,
} from "@selfxyz/qrcode";
import { ethers } from "ethers";

function VerificationPage() {
  const [selfApp, setSelfApp] = useState<SelfApp | null>(null);
  const [universalLink, setUniversalLink] = useState("");
  const [userId] = useState(ethers.ZeroAddress);

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
          /* 1. what you want to verify from users' identity */
          minimumAge: 18,
          // ofac: false,
          // excludedCountries: [countries.BELGIUM],

          /* 2. what you want users to reveal */
          // name: false,
          // issuing_state: true,
          nationality: true,
          // date_of_birth: true,
          // passport_number: false,
          gender: true,
          // expiry_date: false,
        }
      }).build();

      setSelfApp(app);
      setUniversalLink(getUniversalLink(app));
    } catch (error) {
      console.error("Failed to initialize Self app:", error);
    }
  }, [userId]);

  const handleSuccessfulVerification = () => {
    console.log("Verification successful!");
    // Handle success - redirect, update UI, etc.
  };

  return (
    <div className="verification-container">
      <h1>Verify Your Identity</h1>
      <p>Scan this QR code with the Self app</p>
      
      {selfApp ? (
        <SelfQRcodeWrapper
          selfApp={selfApp}
          onSuccess={handleSuccessfulVerification}
          onError={() => {
            console.error("Error: Failed to verify identity");
          }}
        />
      ) : (
        <div>Loading QR Code...</div>
      )}
    </div>
  );
}

export default VerificationPage;
```

### Universal Links (Optional)

The `getUniversalLink` function from `@selfxyz/core` generates deep links that allow users to open the Self app directly instead of scanning a QR code. This is useful for:

- **Mobile web experiences**: Users can tap a button to open the Self app directly
- **Better UX on mobile**: Avoid the need to scan QR codes on the same device
- **Share links**: Send verification links via messaging or email

**When to use Universal Links:**
- Building mobile-first applications
- Want to provide alternative to QR code scanning
- Need to integrate with messaging platforms or email

**Example implementation:**

```javascript
import { getUniversalLink } from "@selfxyz/core";

function VerificationPage() {
  const [selfApp, setSelfApp] = useState(null);
  const [universalLink, setUniversalLink] = useState("");

  useEffect(() => {
    const app = new SelfAppBuilder({...}).build();
    setSelfApp(app);
    
    // Generate universal link for direct app opening
    setUniversalLink(getUniversalLink(app));
  }, []);

  const openSelfApp = () => {
    if (universalLink) {
      window.open(universalLink, "_blank");
    }
  };

  return (
    <div>
      {/* QR Code for desktop/cross-device */}
      <SelfQRcodeWrapper selfApp={selfApp} />
      
      {/* Universal Link button for mobile */}
      <button onClick={openSelfApp}>
        Open Self App
      </button>
    </div>
  );
}
```

> **Note**: Universal links work best on mobile devices where the Self app is installed. On desktop, users should use the QR code method.

### Example

For a more comprehensive and interactive example, please refer to the [playground](https://github.com/selfxyz/playground).

### Verification Flow

The QR code component displays the current verification status with an LED indicator and changes its appearance based on the verification state:

1. **QR Code Display**: Component shows QR code for users to scan
2. **User Scans**: User scans with Self app and provides proof
3. **Backend Verification**: Your API endpoint receives and verifies the proof
4. **Success Callback**: `onSuccess` callback is triggered when verification completes

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

The setup follows a simplified pattern. Here's a complete example based on our workshop:

```javascript
import {
  SelfBackendVerifier,
  AllIds,
  DefaultConfigStore,
  VerificationConfig
} from '@selfxyz/core';

// Define your verification requirements
const verification_config = {
  excludedCountries: [],
  ofac: false,
  minimumAge: 18,
};

// Create the configuration store
const configStore = new DefaultConfigStore(verification_config);

// Initialize the verifier
const selfBackendVerifier = new SelfBackendVerifier(
  "your-app-scope",                           // Your app's unique scope
  "https://your-api-endpoint.com/api/verify", // Your API endpoint
  true,                                       // true = mock for testing, false = production
  AllIds,                                     // Accept all document types
  configStore,                                // Configuration store
  "hex"                                       // "hex" for addresses, "uuid" for UUIDs
);
```

{% hint style="warning" %}
The endpoint must be publicly accessible (not localhost). For local development, use ngrok to tunnel your localhost endpoint.
{% endhint %}

### Verification

The verification process handles the proof data from your frontend:

```javascript
// Extract data from the request
const { attestationId, proof, publicSignals, userContextData } = await req.json();

// Verify all required fields are present
if (!proof || !publicSignals || !attestationId || !userContextData) {
  return NextResponse.json({
    message: "Proof, publicSignals, attestationId and userContextData are required",
  }, { status: 400 });
}

// Verify the proof
const result = await selfBackendVerifier.verify(
  attestationId,    // Document type (1 = passport, 2 = EU ID card)
  proof,            // The zero-knowledge proof
  publicSignals,    // Public signals array
  userContextData   // User context data
);

// Check if verification was successful
if (result.isValidDetails.isValid) {
  // Verification successful - process the result
  return NextResponse.json({
    status: "success",
    result: true,
    credentialSubject: result.discloseOutput,
  });
} else {
  // Verification failed
  return NextResponse.json({
    status: "error",
    result: false,
    message: "Verification failed",
    details: result.isValidDetails,
  }, { status: 500 });
}
```

## Key Points

### Configuration Matching
Your frontend and backend configurations must match exactly:

```javascript
// Backend configuration
const verification_config = {
  excludedCountries: [],
  ofac: false,
  minimumAge: 18,
};

// Frontend configuration (must match)
disclosures: {
  minimumAge: 18,        // Same as backend
  excludedCountries: [], // Same as backend  
  ofac: false,           // Same as backend
  // Plus any disclosure fields you want
  nationality: true,
  gender: true,
}
```