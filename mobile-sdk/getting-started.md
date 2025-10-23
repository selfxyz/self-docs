# Mobile SDK Alpha - Getting Started

The Self Mobile SDK Alpha provides React Native screen components and client infrastructure for integrating Self's identity verification flows into your mobile application.

## Installation

```bash
npm install @selfxyz/mobile-sdk-alpha
```

## Native Modules Setup

⚠️ **Important**: The Mobile SDK requires native module configuration for both Android and iOS platforms. Before using the SDK, you must complete the native setup:

- **[Native Modules Setup](native-modules-setup.md)** - Complete Android and iOS native configuration
- **Android**: MainActivity configuration, build.gradle setup, permissions

## Quick Start

The SDK requires two main integration points:

1. **SelfClientProvider** - Wraps your app with configured adapters and event listeners
2. **Screen Components** - Individual onboarding screens imported from specific paths

### Minimal Example

```tsx
import React from 'react';
import { View } from 'react-native';
import { 
  SelfClientProvider, 
  createListenersMap,
  type Adapters 
} from '@selfxyz/mobile-sdk-alpha';
import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';

// Configure adapters (see selfclient-provider.md for details)
const adapters: Adapters = {
  auth: {
    getPrivateKey: async () => "your-private-key"
  },
  scanner: yourNFCScannerAdapter,
  network: {
    http: { fetch },
    ws: { connect: (url) => new WebSocket(url) }
  },
  crypto: {
    hash: async (data) => /* hash implementation */,
    sign: async (data, keyRef) => /* signing implementation */
  },
  documents: yourDocumentStorageAdapter
};

// Configure event listeners for navigation
const listeners = createListenersMap();
const config = {};

function App() {
  return (
    <SelfClientProvider 
      config={config} 
      adapters={adapters} 
      listeners={listeners.map}
    >
      <View style={{ flex: 1 }}>
        <DocumentCameraScreen 
          onBack={() => console.log('Go back')}
          onSuccess={() => console.log('MRZ scan successful')}
        />
      </View>
    </SelfClientProvider>
  );
}
```

## Next Steps

- **[Native Modules Setup](native-modules-setup.md)** - Configure Android and iOS native modules
- **[SelfClient Provider Setup](selfclient-provider.md)** - Configure adapters and event listeners
- **[Onboarding Screen Components](onboarding-screens.md)** - Use the available screen components
- **[Implementation Examples](examples/README.md)** - See complete integration patterns

## Important Notes

⚠️ **Alpha Status**: This SDK is in alpha. The interface may change as it matures. Many current configuration options may be removed or simplified in future versions.

✅ **Core Stability**: The two main integration patterns (SelfClientProvider + screen components) are expected to remain stable.