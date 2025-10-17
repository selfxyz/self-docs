# SelfClient Provider Setup

The `SelfClientProvider` is a React context provider that must wrap all screens using the Mobile SDK. It configures the SDK with the necessary adapters and event listeners your app needs.

## Basic Setup

```tsx
import { 
  SelfClientProvider, 
  createListenersMap,
  type Adapters,
  type Config 
} from '@selfxyz/mobile-sdk-alpha';

const config: Config = {};

const adapters: Adapters = {
  // Required adapters
  auth: authAdapter,
  scanner: scannerAdapter, 
  network: networkAdapter,
  crypto: cryptoAdapter,
  documents: documentsAdapter,
  
  // Optional adapters (SDK provides defaults)
  analytics: analyticsAdapter,
  logger: loggerAdapter,
  clock: clockAdapter
};

const { map: listeners } = createListenersMap();

function App() {
  return (
    <SelfClientProvider 
      config={config}
      adapters={adapters} 
      listeners={listeners}
    >
      {/* Your app screens here */}
    </SelfClientProvider>
  );
}
```

## Required Adapters

### Auth Adapter

Manages private key access for cryptographic operations.

```tsx
const authAdapter = {
  async getPrivateKey(): Promise<string | null> {
    // Return private key as hex string (with or without 0x prefix)
    // Return null if no key available
    return await yourSecureStorage.getPrivateKey();
  }
};
```

### Scanner Adapter  

Provides NFC scanning capabilities.

```tsx
// For React Native
import { reactNativeScannerAdapter } from '@selfxyz/mobile-sdk-alpha';
const scannerAdapter = reactNativeScannerAdapter;

// For Web (development/testing)
import { webNFCScannerShim } from '@selfxyz/mobile-sdk-alpha';
const scannerAdapter = webNFCScannerShim;
```

### Network Adapter

Handles HTTP and WebSocket communications.

```tsx
const networkAdapter = {
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
};
```

### Crypto Adapter

Provides hashing and signing operations.

```tsx
const cryptoAdapter = {
  async hash(data: Uint8Array, algo: 'sha256' = 'sha256'): Promise<Uint8Array> {
    // Use Web Crypto API or native crypto library
    const buffer = await crypto.subtle.digest('SHA-256', data);
    return new Uint8Array(buffer);
  },
  
  async sign(data: Uint8Array, keyRef: string): Promise<Uint8Array> {
    // Implement signing logic for your key management
    throw new Error('Signing not implemented');
  }
};
```

### Documents Adapter

Handles document storage and retrieval.

```tsx
const documentsAdapter = {
  async loadDocumentCatalog(): Promise<DocumentCatalog> {
    return await yourStorage.getCatalog();
  },
  
  async saveDocumentCatalog(catalog: DocumentCatalog): Promise<void> {
    await yourStorage.saveCatalog(catalog);
  },
  
  async loadDocumentById(id: string): Promise<IDDocument | null> {
    return await yourStorage.getDocument(id);
  },
  
  async saveDocument(id: string, document: IDDocument): Promise<void> {
    await yourStorage.saveDocument(id, document);
  },
  
  async deleteDocument(id: string): Promise<void> {
    await yourStorage.deleteDocument(id);
  }
};
```

## Event Listeners & Navigation

Event listeners handle SDK events and route to appropriate screens in your app.

```tsx
import { SdkEvents } from '@selfxyz/mobile-sdk-alpha';

const { map: listeners, addListener } = createListenersMap();

// Handle country selection -> navigate to ID picker
addListener(SdkEvents.DOCUMENT_COUNTRY_SELECTED, ({ countryCode, documentTypes }) => {
  navigation.navigate('IDPicker', { countryCode, documentTypes });
});

// Handle document type selection -> navigate to appropriate flow
addListener(SdkEvents.DOCUMENT_TYPE_SELECTED, ({ documentType, countryCode }) => {
  switch (documentType) {
    case 'p': // Passport
    case 'i': // ID Card
      navigation.navigate('DocumentCamera');
      break;
    case 'a': // Aadhaar
      navigation.navigate('AadhaarUpload', { countryCode });
      break;
    default:
      navigation.navigate('ComingSoon', { countryCode });
  }
});

// Handle successful MRZ scan -> navigate to NFC scan
addListener(SdkEvents.DOCUMENT_MRZ_READ_SUCCESS, () => {
  navigation.navigate('NFCScan');
});

// Handle MRZ scan failure -> navigate to troubleshooting
addListener(SdkEvents.DOCUMENT_MRZ_READ_FAILURE, () => {
  navigation.navigate('DocumentTrouble');
});
```

## Common SDK Events

| Event | Payload | Description |
|-------|---------|-------------|
| `DOCUMENT_COUNTRY_SELECTED` | `{ countryCode, documentTypes }` | User selected a country |
| `DOCUMENT_TYPE_SELECTED` | `{ documentType, countryCode }` | User selected document type |
| `DOCUMENT_MRZ_READ_SUCCESS` | `{}` | MRZ scanning completed successfully |
| `DOCUMENT_MRZ_READ_FAILURE` | `{}` | MRZ scanning failed |