# Minimal Setup Example

This example shows the bare minimum integration required to use the Mobile SDK screens.

## Complete Example

```tsx
import React, { useMemo } from 'react';
import { View, SafeAreaView } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

import {
  SelfClientProvider,
  createListenersMap,
  SdkEvents,
  webNFCScannerShim,
  type Adapters,
  type Config
} from '@selfxyz/mobile-sdk-alpha';

import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';
import IDSelectionScreen from '@selfxyz/mobile-sdk-alpha/onboarding/id-selection-screen';
import SDKCountryPickerScreen from '@selfxyz/mobile-sdk-alpha/onboarding/country-picker-screen';

const Stack = createStackNavigator();

// Minimal adapter implementations
const createAdapters = (): Adapters => ({
  auth: {
    async getPrivateKey(): Promise<string | null> {
      // In production, get from secure storage
      return "0x" + "a".repeat(64); // Dummy key for demo
    }
  },

  scanner: webNFCScannerShim, // Use web shim for development

  network: {
    http: {
      fetch: (input: RequestInfo, init?: RequestInit) => fetch(input, init)
    },
    ws: {
      connect: (url: string) => {
        const socket = new WebSocket(url);
        return {
          send: (data) => socket.send(data),
          close: () => socket.close(),
          onMessage: (cb) => socket.addEventListener('message', ev => cb(ev.data)),
          onError: (cb) => socket.addEventListener('error', cb),
          onClose: (cb) => socket.addEventListener('close', cb)
        };
      }
    }
  },

  crypto: {
    async hash(data: Uint8Array): Promise<Uint8Array> {
      // Use Web Crypto API
      const buffer = await crypto.subtle.digest('SHA-256', data);
      return new Uint8Array(buffer);
    },
    async sign(_data: Uint8Array, _keyRef: string): Promise<Uint8Array> {
      throw new Error('Signing not implemented in minimal example');
    }
  },

  documents: {
    async loadDocumentCatalog() {
      return { documents: [] };
    },
    async saveDocumentCatalog(catalog) {
      console.log('Save catalog:', catalog);
    },
    async loadDocumentById(id: string) {
      return null;
    },
    async saveDocument(id: string, document) {
      console.log('Save document:', id, document);
    },
    async deleteDocument(id: string) {
      console.log('Delete document:', id);
    }
  },

  // Optional: minimal analytics
  analytics: {
    trackEvent: (event: string, payload?: any) => {
      console.log('Analytics:', event, payload);
    }
  }
});

// Screen components
function CountryPickerScreen({ navigation }) {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      <SDKCountryPickerScreen />
    </SafeAreaView>
  );
}

function IDPickerScreen({ navigation, route }) {
  const { countryCode, documentTypes } = route.params;
  
  return (
    <SafeAreaView style={{ flex: 1 }}>
      <IDSelectionScreen 
        countryCode={countryCode}
        documentTypes={documentTypes}
      />
    </SafeAreaView>
  );
}

function DocumentCameraStackScreen({ navigation }) {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      <DocumentCameraScreen
        onBack={() => navigation.goBack()}
        onSuccess={() => {
          console.log('MRZ scan successful!');
          alert('Document scanned successfully!');
        }}
      />
    </SafeAreaView>
  );
}

// Main app with provider and navigation setup
function App() {
  const config: Config = useMemo(() => ({}), []);
  const adapters: Adapters = useMemo(() => createAdapters(), []);

  const listeners = useMemo(() => {
    const { map, addListener } = createListenersMap();

    // Handle country selection
    addListener(SdkEvents.DOCUMENT_COUNTRY_SELECTED, ({ countryCode, documentTypes }) => {
      navigation.navigate('IDPicker', { countryCode, documentTypes });
    });

    // Handle document type selection  
    addListener(SdkEvents.DOCUMENT_TYPE_SELECTED, ({ documentType }) => {
      if (documentType === 'p' || documentType === 'i') {
        navigation.navigate('DocumentCamera');
      } else {
        alert(`Document type ${documentType} not supported in this example`);
      }
    });

    // Handle successful MRZ scan
    addListener(SdkEvents.DOCUMENT_MRZ_READ_SUCCESS, () => {
      console.log('MRZ read success event received');
    });

    // Handle MRZ scan failure
    addListener(SdkEvents.DOCUMENT_MRZ_READ_FAILURE, () => {
      console.log('MRZ read failure event received');
      alert('Document scan failed. Please try again.');
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
        <Stack.Navigator initialRouteName="CountryPicker">
          <Stack.Screen 
            name="CountryPicker" 
            component={CountryPickerScreen}
            options={{ title: 'Select Country' }}
          />
          <Stack.Screen 
            name="IDPicker" 
            component={IDPickerScreen}
            options={{ title: 'Select Document Type' }}
          />
          <Stack.Screen 
            name="DocumentCamera" 
            component={DocumentCameraStackScreen}
            options={{ title: 'Scan Document' }}
          />
        </Stack.Navigator>
      </NavigationContainer>
    </SelfClientProvider>
  );
}

export default App;
```

## Key Points

### 1. Provider Wrapping
```tsx
<SelfClientProvider config={config} adapters={adapters} listeners={listeners}>
  {/* All screens must be inside this provider */}
</SelfClientProvider>
```

### 2. Direct Screen Imports
```tsx
import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';
import IDSelectionScreen from '@selfxyz/mobile-sdk-alpha/onboarding/id-selection-screen';
```

### 3. Event-Driven Navigation
```tsx
addListener(SdkEvents.DOCUMENT_COUNTRY_SELECTED, ({ countryCode, documentTypes }) => {
  navigation.navigate('IDPicker', { countryCode, documentTypes });
});
```

### 4. Required Adapters
All five required adapters must be provided:
- `auth` - Private key access
- `scanner` - NFC scanning capability  
- `network` - HTTP/WebSocket connections
- `crypto` - Hashing and signing
- `documents` - Document persistence

This minimal example provides a complete working integration that you can build upon.