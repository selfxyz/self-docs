# Demo App Walkthrough

This walkthrough explains how the `mobile-sdk-demo` package implements the SDK integration patterns.

## Project Structure

```
packages/mobile-sdk-demo/
├── App.tsx                 # Main app with provider setup
├── src/
│   ├── providers/
│   │   └── SelfClientProvider.tsx  # SDK provider configuration
│   ├── screens/            # Screen wrappers
│   │   ├── CountrySelection.tsx
│   │   ├── IDSelection.tsx  
│   │   └── DocumentCamera.tsx
│   └── navigation/         # Navigation logic
```

## Provider Setup

The demo shows how to configure adapters for different environments:

```tsx
// src/providers/SelfClientProvider.tsx
const adapters: Adapters = {
  scanner: webNFCScannerShim, // Web environment
  network: {
    http: { fetch: createFetch() },
    ws: createWsAdapter()
  },
  documents: persistentDocumentsAdapter,
  crypto: { hash, sign: stubSign },
  auth: { getPrivateKey: getOrCreateSecret }
};
```

## Screen Wrapper Pattern

The demo wraps SDK screens in custom components:

```tsx
// src/screens/DocumentCamera.tsx
import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';

export default function DocumentCamera({ onBack, onSuccess }) {
  return <DocumentCameraScreen onBack={onBack} onSuccess={onSuccess} />;
}
```

## Event Listener Setup

```tsx
// src/providers/SelfClientProvider.tsx
const listeners = useMemo(() => {
  const { map, addListener } = createListenersMap();

  addListener(SdkEvents.DOCUMENT_COUNTRY_SELECTED, event => {
    navigation.navigate('IDSelection', {
      countryCode: event.countryCode,
      countryName: event.countryName,
      documentTypes: event.documentTypes,
    });
  });

  return map;
}, [navigation]);
```

## Key Patterns

### 1. Environment Adaptation
The demo handles both web and mobile environments by choosing appropriate adapters.

### 2. Custom Navigation
Uses React Navigation with custom screen wrappers that pass navigation props to SDK screens.

### 3. State Management  
Manages document catalog and selected document state at the app level.

### 4. Error Handling
Basic error handling with console warnings and fallback states.

This demonstrates a practical, working integration you can reference for your own implementation.