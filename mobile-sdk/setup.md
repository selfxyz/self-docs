# SelfClientProvider Setup

The `SelfClientProvider` is the core component that wraps your React Native app and provides access to the Self SDK functionality. This guide shows you how to configure the provider with adapters and event listeners.

## Basic Configuration

```typescript
import React, { useMemo } from 'react';
import { 
  SelfClientProvider, 
  createListenersMap,
  reactNativeScannerAdapter,
  SdkEvents 
} from '@selfxyz/mobile-sdk-alpha';
import { NavigationContainer } from '@react-navigation/native';

export default function App() {
  // Configuration (can be empty for defaults)
  const config = useMemo(() => ({}), []);
  
  // Setup adapters (see adapters.md for implementations)
  const adapters = useMemo(() => ({
    // Required adapters
    scanner: reactNativeScannerAdapter,
    crypto: cryptoAdapter,
    network: networkAdapter, 
    auth: authAdapter,
    documents: documentsAdapter,
    
    // Optional adapters
    analytics: analyticsAdapter,
    logger: loggerAdapter,
    storage: storageAdapter,
  }), []);
  
  // Setup event listeners
  const listeners = useMemo(() => {
    const { map, addListener } = createListenersMap();
    
    // Add your event listeners here (see Event Listeners section below)
    addListener(SdkEvents.PROVING_PASSPORT_DATA_NOT_FOUND, () => {
      // Handle navigation when no passport data found
    });
    
    return map;
  }, []);

  return (
    <SelfClientProvider 
      config={config} 
      adapters={adapters} 
      listeners={listeners}
    >
      <NavigationContainer>
        {/* Your app content */}
      </NavigationContainer>
    </SelfClientProvider>
  );
}
```

## Configuration Options

The `config` parameter accepts these optional settings:

```typescript
const config = {
  endpoints: {
    api: 'https://custom-api.self.id',           // API endpoint
    teeWs: 'wss://custom-tee.self.id',          // TEE WebSocket
    artifactsCdn: 'https://custom-cdn.self.id'  // Artifacts CDN
  },
  timeouts: {
    httpMs: 30000,    // HTTP request timeout
    wsMs: 30000,      // WebSocket timeout  
    scanMs: 60000,    // Scanning timeout
    proofMs: 300000   // Proof generation timeout
  },
  features: {
    'feature-flag-name': true  // Enable/disable features
  },
  tlsPinning: {
    enabled: true,
    pins: ['sha256-pin1', 'sha256-pin2']  // Certificate pins
  }
};
```

## Required Adapters Overview

You must provide these five adapters for the SDK to function:

### Scanner Adapter
```typescript
import { reactNativeScannerAdapter } from '@selfxyz/mobile-sdk-alpha';

const adapters = {
  scanner: reactNativeScannerAdapter, // Use the provided implementation
  // ...
};
```

**Purpose:** Handles MRZ and NFC document scanning via native modules.

### Network Adapter  
```typescript
const networkAdapter = {
  http: { fetch: (input, init) => fetch(input, init) },
  ws: { connect: (url) => new WebSocket(url) }
};
```

**Purpose:** Manages HTTP requests and WebSocket connections to Self services.

### Crypto Adapter
```typescript
const cryptoAdapter = {
  async hash(data, algo = 'sha256') { /* Web Crypto API */ },
  async sign(data, keyRef) { /* Signing implementation */ }
};
```

**Purpose:** Provides cryptographic operations for secure communications.

### Auth Adapter
```typescript
const authAdapter = {
  async getPrivateKey() { /* Keychain access */ }
};
```

**Purpose:** Securely manages and retrieves the user's private key.

### Documents Adapter
```typescript
const documentsAdapter = {
  async loadDocumentCatalog() { /* Load from secure storage */ },
  async saveDocument(id, data) { /* Save to secure storage */ },
  // ... other methods
};
```

**Purpose:** Handles secure storage and retrieval of passport/document data.

For detailed implementations of each adapter, see the **[Adapters Guide](adapters.md)**.

## Event Listeners

The SDK emits events that your app should handle for proper user experience:

```typescript
import { createListenersMap, SdkEvents } from '@selfxyz/mobile-sdk-alpha';

const listeners = useMemo(() => {
  const { map, addListener } = createListenersMap();

  // Required: No passport data found - navigate to document setup
  addListener(SdkEvents.PROVING_PASSPORT_DATA_NOT_FOUND, () => {
    navigation.navigate('DocumentSetup');
  });

  // Required: Identity verification successful  
  addListener(SdkEvents.PROVING_ACCOUNT_VERIFIED_SUCCESS, () => {
    setTimeout(() => {
      navigation.navigate('SuccessScreen');
    }, 3000); // Show success state for 3 seconds
  });

  // Required: Handle registration errors
  addListener(SdkEvents.PROVING_REGISTER_ERROR_OR_FAILURE, ({ hasValidDocument }) => {
    setTimeout(() => {
      if (hasValidDocument) {
        navigation.navigate('Home'); // User has other valid documents
      } else {
        navigation.navigate('Onboarding'); // User needs to register documents
      }
    }, 3000);
  });

  // Required: Unsupported document detected
  addListener(SdkEvents.PROVING_PASSPORT_NOT_SUPPORTED, ({ countryCode, documentCategory }) => {
    navigation.navigate('UnsupportedDocument', { countryCode, documentCategory });
  });

  // Required: Account recovery needed
  addListener(SdkEvents.PROVING_ACCOUNT_RECOVERY_REQUIRED, () => {
    navigation.navigate('AccountRecovery');
  });

  // Recommended: Show progress updates
  addListener(SdkEvents.PROGRESS, (progress) => {
    setProgressState(progress);
  });

  // Recommended: Handle errors
  addListener(SdkEvents.ERROR, (error) => {
    showErrorModal(error.message);
  });

  return map;
}, []);
```

### Event Descriptions

| Event | When Emitted | Required Action |
|-------|-------------|----------------|
| `PROVING_PASSPORT_DATA_NOT_FOUND` | No document data found on device | Navigate to document scanning screen |
| `PROVING_ACCOUNT_VERIFIED_SUCCESS` | Identity verification completed | Show success, navigate to main app |
| `PROVING_REGISTER_ERROR_OR_FAILURE` | Document registration failed | Navigate based on `hasValidDocument` flag |
| `PROVING_PASSPORT_NOT_SUPPORTED` | Unsupported document detected | Show unsupported document screen |
| `PROVING_ACCOUNT_RECOVERY_REQUIRED` | Document registered with different credentials | Navigate to account recovery |
| `PROGRESS` | Long-running operation progress | Update progress indicators |
| `ERROR` | SDK operation errors | Show error messages to user |

## Using the SDK in Components

After setting up the provider, use the `useSelfClient()` hook in your components:

```typescript
import { useSelfClient } from '@selfxyz/mobile-sdk-alpha';

function DocumentScanScreen() {
  const selfClient = useSelfClient();
  
  const handleMRZScan = async () => {
    try {
      const result = await selfClient.scanDocument({ mode: 'mrz' });
      // Handle scan result
    } catch (error) {
      // Handle error
    }
  };

  return (
    // Your UI
  );
}
```

## Complete Minimal Example

```typescript
import React, { useMemo } from 'react';
import { 
  SelfClientProvider, 
  createListenersMap,
  reactNativeScannerAdapter,
  SdkEvents 
} from '@selfxyz/mobile-sdk-alpha';

// Import your adapter implementations (see adapters.md)
import { 
  networkAdapter, 
  cryptoAdapter, 
  authAdapter, 
  documentsAdapter 
} from './adapters';

export function App() {
  const config = useMemo(() => ({}), []);
  
  const adapters = useMemo(() => ({
    scanner: reactNativeScannerAdapter,
    network: networkAdapter,
    crypto: cryptoAdapter,
    auth: authAdapter,
    documents: documentsAdapter,
  }), []);

  const listeners = useMemo(() => {
    const { map, addListener } = createListenersMap();
    
    addListener(SdkEvents.PROVING_PASSPORT_DATA_NOT_FOUND, () => {
      // Navigate to document scanning
    });
    
    return map;
  }, []);

  return (
    <SelfClientProvider config={config} adapters={adapters} listeners={listeners}>
      {/* Your app */}
    </SelfClientProvider>
  );
}
```

## Next Steps

1. **[Implement Adapters](adapters.md)** - Create the required adapter implementations
2. **Test Your Setup** - Verify the provider works with mock documents
3. **Add Document Scanning** - Implement MRZ and NFC scanning screens
4. **Handle Proofs** - Add proof generation and verification flows