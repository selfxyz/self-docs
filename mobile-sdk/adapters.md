# Adapter Implementations

This guide provides detailed implementations for the adapters required by the `SelfClientProvider`. Each adapter bridges the SDK with your React Native app's platform-specific functionality.

## Required Adapters

### Scanner Adapter

Use the provided React Native scanner adapter which handles MRZ and NFC scanning via native modules.

```typescript
import { reactNativeScannerAdapter } from '@selfxyz/mobile-sdk-alpha';

const adapters = {
  scanner: reactNativeScannerAdapter,
  // ... other adapters
};
```

**Implementation Details:**
- Interfaces with native iOS and Android scanner modules
- Supports MRZ (machine-readable zone) scanning
- Handles NFC passport chip reading
- Requires proper native module linking

**Native Module Requirements:**
- iOS: `SelfMRZScannerModule`, `PassportReader`  
- Android: `SelfMRZScannerModule`, `SelfPassportReader`

### Network Adapter

Handles HTTP requests and WebSocket connections to Self services.

```typescript
import type { NetworkAdapter, WsConn } from '@selfxyz/mobile-sdk-alpha';

const networkAdapter: NetworkAdapter = {
  http: {
    fetch: (input: RequestInfo, init?: RequestInit) => fetch(input, init),
  },
  ws: {
    connect: (url: string): WsConn => {
      const socket = new WebSocket(url);
      return {
        send: (data: string | ArrayBufferView | ArrayBuffer) => socket.send(data),
        close: () => socket.close(),
        onMessage: (cb) => {
          socket.addEventListener('message', (ev) => cb((ev as MessageEvent).data));
        },
        onError: (cb) => {
          socket.addEventListener('error', (e) => cb(e));
        },
        onClose: (cb) => {
          socket.addEventListener('close', () => cb());
        },
      };
    },
  },
};
```

**Why needed:** Communicates with Self's API services for proof generation and verification.

**Suggested enhancements:**
```typescript
// Add timeout and retry logic for production
const networkAdapter: NetworkAdapter = {
  http: {
    fetch: async (input: RequestInfo, init?: RequestInit) => {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 30000); // 30s timeout
      
      try {
        const response = await fetch(input, {
          ...init,
          signal: controller.signal,
        });
        clearTimeout(timeoutId);
        return response;
      } catch (error) {
        clearTimeout(timeoutId);
        throw error;
      }
    },
  },
  // ... ws implementation
};
```

### Crypto Adapter

Provides cryptographic operations using Web Crypto API or polyfills.

```typescript
import type { CryptoAdapter } from '@selfxyz/mobile-sdk-alpha';

const cryptoAdapter: CryptoAdapter = {
  async hash(data: Uint8Array, algo: 'sha256' = 'sha256'): Promise<Uint8Array> {
    // Use Web Crypto API if available
    const subtle = (globalThis as any)?.crypto?.subtle;
    if (!subtle?.digest) {
      throw new Error('WebCrypto not available; provide crypto polyfill for React Native');
    }
    
    const webCryptoAlgo = algo === 'sha256' ? 'SHA-256' : algo;
    const buf = await subtle.digest(webCryptoAlgo, data as BufferSource);
    return new Uint8Array(buf);
  },

  async sign(_data: Uint8Array, _keyRef: string): Promise<Uint8Array> {
    // Implement signing logic if needed for your use case
    throw new Error(`crypto.sign not implemented for keyRef: ${_keyRef}`);
  },
};
```

**Alternative with crypto polyfill:**
```typescript
import { sha256 } from '@noble/hashes/sha256';

const cryptoAdapter: CryptoAdapter = {
  async hash(data: Uint8Array, algo: 'sha256' = 'sha256'): Promise<Uint8Array> {
    if (algo === 'sha256') {
      return sha256(data);
    }
    throw new Error(`Unsupported hash algorithm: ${algo}`);
  },
  
  async sign(_data: Uint8Array, _keyRef: string): Promise<Uint8Array> {
    throw new Error(`crypto.sign not implemented for keyRef: ${_keyRef}`);
  },
};
```

### Auth Adapter

Manages secure private key storage and retrieval using device keychain/keystore.

```typescript
import type { AuthAdapter } from '@selfxyz/mobile-sdk-alpha';
import Keychain from 'react-native-keychain';

const authAdapter: AuthAdapter = {
  async getPrivateKey(): Promise<string | null> {
    try {
      const credentials = await Keychain.getGenericPassword({ 
        service: 'user-private-key',
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
      });
      
      if (credentials !== false) {
        // Return hex-encoded private key
        return credentials.password;
      }
      return null;
    } catch (error) {
      console.error('Failed to retrieve private key:', error);
      return null;
    }
  }
};
```

**Enhanced implementation with key generation:**
```typescript
import { ethers } from 'ethers';
import Keychain from 'react-native-keychain';

const authAdapter: AuthAdapter = {
  async getPrivateKey(): Promise<string | null> {
    try {
      // Try to load existing key
      const credentials = await Keychain.getGenericPassword({ 
        service: 'user-private-key',
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
      });
      
      if (credentials !== false) {
        return credentials.password;
      }
      
      // Generate new key if none exists
      const wallet = ethers.Wallet.createRandom();
      const privateKey = wallet.privateKey;
      
      // Store securely
      await Keychain.setGenericPassword(
        'user-private-key',
        privateKey,
        {
          service: 'user-private-key',
          accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
        }
      );
      
      return privateKey;
    } catch (error) {
      console.error('Auth adapter error:', error);
      return null;
    }
  }
};
```

**Security considerations:**
- Use biometric access control when possible
- Store keys in hardware security modules if available  
- Never log or expose private keys
- Consider mnemonic-based recovery

### Documents Adapter

Handles secure storage of passport/document data with content deduplication.

```typescript
import type { DocumentsAdapter, DocumentCatalog, PassportData } from '@selfxyz/mobile-sdk-alpha';
import Keychain from 'react-native-keychain';

const documentsAdapter: DocumentsAdapter = {
  async loadDocumentCatalog(): Promise<DocumentCatalog> {
    try {
      const credentials = await Keychain.getGenericPassword({ 
        service: 'document-catalog' 
      });
      
      if (credentials !== false) {
        return JSON.parse(credentials.password);
      }
    } catch (error) {
      console.warn('Failed to load document catalog:', error);
    }
    
    // Return empty catalog if none exists
    return { documents: [] };
  },

  async saveDocumentCatalog(catalog: DocumentCatalog): Promise<void> {
    await Keychain.setGenericPassword(
      'catalog', 
      JSON.stringify(catalog), 
      { service: 'document-catalog' }
    );
  },

  async loadDocumentById(id: string): Promise<PassportData | null> {
    try {
      const credentials = await Keychain.getGenericPassword({ 
        service: `document-${id}` 
      });
      
      if (credentials !== false) {
        return JSON.parse(credentials.password);
      }
    } catch (error) {
      console.warn(`Failed to load document ${id}:`, error);
    }
    return null;
  },

  async saveDocument(id: string, passportData: PassportData): Promise<void> {
    await Keychain.setGenericPassword(
      id, 
      JSON.stringify(passportData), 
      { service: `document-${id}` }
    );
  },

  async deleteDocument(id: string): Promise<void> {
    await Keychain.resetGenericPassword({ service: `document-${id}` });
  }
};
```

**Storage Architecture:**
- **Master Catalog**: `document-catalog` contains metadata for all documents
- **Individual Documents**: `document-{uuid}` stores actual passport data
- **Content Deduplication**: Uses content hashes to avoid duplicate storage

**Enhanced implementation with migration support:**
```typescript
const documentsAdapter: DocumentsAdapter = {
  async loadDocumentCatalog(): Promise<DocumentCatalog> {
    try {
      const credentials = await Keychain.getGenericPassword({ 
        service: 'document-catalog' 
      });
      
      if (credentials !== false) {
        const catalog = JSON.parse(credentials.password);
        
        // Handle migration from older versions if needed
        if (!catalog.version) {
          catalog.version = '1.0';
          await this.saveDocumentCatalog(catalog);
        }
        
        return catalog;
      }
    } catch (error) {
      console.warn('Failed to load document catalog:', error);
    }
    
    return { documents: [], version: '1.0' };
  },
  
  // ... other methods remain the same
};
```

## Optional Adapters

### Analytics Adapter

Integrates with your analytics service for tracking SDK events.

```typescript
import type { AnalyticsAdapter, TrackEventParams } from '@selfxyz/mobile-sdk-alpha';

const analyticsAdapter: AnalyticsAdapter = {
  trackEvent(event: string, payload?: TrackEventParams): void {
    // Example: Integrate with Segment
    // analytics.track(event, payload);
    
    // Example: Custom analytics
    console.log('Analytics event:', event, payload);
    
    // Example: Multiple services
    // mixpanel.track(event, payload);
    // amplitude.logEvent(event, payload);
  }
};
```

### Logger Adapter

Routes SDK logs to your logging service.

```typescript
import type { LoggerAdapter, LogLevel } from '@selfxyz/mobile-sdk-alpha';

const loggerAdapter: LoggerAdapter = {
  log(level: LogLevel, message: string, fields?: Record<string, unknown>): void {
    // Console logging
    console[level](`[SDK] ${message}`, fields);
    
    // Example: Send to external logging service
    // if (level === 'error') {
    //   Sentry.addBreadcrumb({ message, level, data: fields });
    // }
  }
};
```

### Storage Adapter

General purpose key-value storage for SDK preferences.

```typescript
import type { StorageAdapter } from '@selfxyz/mobile-sdk-alpha';
import AsyncStorage from '@react-native-async-storage/async-storage';

const storageAdapter: StorageAdapter = {
  async get(key: string): Promise<string | null> {
    try {
      return await AsyncStorage.getItem(key);
    } catch (error) {
      console.warn(`Storage get error for key ${key}:`, error);
      return null;
    }
  },

  async set(key: string, value: string): Promise<void> {
    try {
      await AsyncStorage.setItem(key, value);
    } catch (error) {
      console.error(`Storage set error for key ${key}:`, error);
      throw error;
    }
  },

  async remove(key: string): Promise<void> {
    try {
      await AsyncStorage.removeItem(key);
    } catch (error) {
      console.error(`Storage remove error for key ${key}:`, error);
      throw error;
    }
  }
};
```

## Complete Adapters Implementation

Create an `adapters.ts` file to export all your adapters:

```typescript
// adapters.ts
import type {
  NetworkAdapter,
  CryptoAdapter,
  AuthAdapter,
  DocumentsAdapter,
  AnalyticsAdapter,
  LoggerAdapter,
  StorageAdapter,
} from '@selfxyz/mobile-sdk-alpha';
import { reactNativeScannerAdapter } from '@selfxyz/mobile-sdk-alpha';
import Keychain from 'react-native-keychain';
import AsyncStorage from '@react-native-async-storage/async-storage';

// Network Adapter
export const networkAdapter: NetworkAdapter = {
  http: { 
    fetch: (input, init) => fetch(input, init) 
  },
  ws: {
    connect: (url) => {
      const socket = new WebSocket(url);
      return {
        send: (data) => socket.send(data),
        close: () => socket.close(),
        onMessage: (cb) => socket.addEventListener('message', (ev) => cb(ev.data)),
        onError: (cb) => socket.addEventListener('error', cb),
        onClose: (cb) => socket.addEventListener('close', cb),
      };
    }
  }
};

// Crypto Adapter
export const cryptoAdapter: CryptoAdapter = {
  async hash(data, algo = 'sha256') {
    const subtle = globalThis.crypto?.subtle;
    if (!subtle) throw new Error('WebCrypto not available');
    const buf = await subtle.digest('SHA-256', data);
    return new Uint8Array(buf);
  },
  async sign() { 
    throw new Error('Sign not implemented'); 
  }
};

// Auth Adapter
export const authAdapter: AuthAdapter = {
  async getPrivateKey() {
    try {
      const creds = await Keychain.getGenericPassword({ 
        service: 'user-private-key' 
      });
      return creds !== false ? creds.password : null;
    } catch (error) {
      console.error('Failed to get private key:', error);
      return null;
    }
  }
};

// Documents Adapter
export const documentsAdapter: DocumentsAdapter = {
  async loadDocumentCatalog() {
    try {
      const creds = await Keychain.getGenericPassword({ 
        service: 'document-catalog' 
      });
      return creds !== false ? JSON.parse(creds.password) : { documents: [] };
    } catch (error) {
      console.warn('Failed to load catalog:', error);
      return { documents: [] };
    }
  },

  async saveDocumentCatalog(catalog) {
    await Keychain.setGenericPassword(
      'catalog', 
      JSON.stringify(catalog), 
      { service: 'document-catalog' }
    );
  },

  async loadDocumentById(id) {
    try {
      const creds = await Keychain.getGenericPassword({ 
        service: `document-${id}` 
      });
      return creds !== false ? JSON.parse(creds.password) : null;
    } catch (error) {
      console.warn(`Failed to load document ${id}:`, error);
      return null;
    }
  },

  async saveDocument(id, data) {
    await Keychain.setGenericPassword(
      id, 
      JSON.stringify(data), 
      { service: `document-${id}` }
    );
  },

  async deleteDocument(id) {
    await Keychain.resetGenericPassword({ service: `document-${id}` });
  }
};

// Optional: Analytics Adapter
export const analyticsAdapter: AnalyticsAdapter = {
  trackEvent(event, payload) {
    console.log('Analytics:', event, payload);
  }
};

// Optional: Logger Adapter
export const loggerAdapter: LoggerAdapter = {
  log(level, message, fields) {
    console[level](`[SDK] ${message}`, fields);
  }
};

// Optional: Storage Adapter
export const storageAdapter: StorageAdapter = {
  async get(key) {
    return AsyncStorage.getItem(key);
  },
  async set(key, value) {
    await AsyncStorage.setItem(key, value);
  },
  async remove(key) {
    await AsyncStorage.removeItem(key);
  }
};

// Export scanner adapter
export const scannerAdapter = reactNativeScannerAdapter;
```

## Usage in SelfClientProvider

```typescript
import React, { useMemo } from 'react';
import { SelfClientProvider, createListenersMap } from '@selfxyz/mobile-sdk-alpha';
import {
  scannerAdapter,
  networkAdapter,
  cryptoAdapter, 
  authAdapter,
  documentsAdapter,
  analyticsAdapter,
  loggerAdapter,
  storageAdapter,
} from './adapters';

export function App() {
  const adapters = useMemo(() => ({
    scanner: scannerAdapter,
    network: networkAdapter,
    crypto: cryptoAdapter,
    auth: authAdapter,
    documents: documentsAdapter,
    analytics: analyticsAdapter,
    logger: loggerAdapter,
    storage: storageAdapter,
  }), []);

  // ... rest of setup

  return (
    <SelfClientProvider adapters={adapters} /* ... other props */>
      {/* Your app */}
    </SelfClientProvider>
  );
}
```

## Testing Your Adapters

Test each adapter individually before using them with the SDK:

```typescript
// Test documents adapter
const testDocumentsAdapter = async () => {
  const catalog = await documentsAdapter.loadDocumentCatalog();
  console.log('Catalog loaded:', catalog);
};

// Test auth adapter  
const testAuthAdapter = async () => {
  const key = await authAdapter.getPrivateKey();
  console.log('Key exists:', !!key);
};

// Test crypto adapter
const testCryptoAdapter = async () => {
  const data = new Uint8Array([1, 2, 3, 4]);
  const hash = await cryptoAdapter.hash(data);
  console.log('Hash generated:', hash.length === 32);
};
```

## Next Steps

1. **[Return to Setup Guide](setup.md)** - Configure your SelfClientProvider
2. **Test Integration** - Verify adapters work with the SDK
3. **Implement Screens** - Add document scanning and verification UI
4. **Handle Edge Cases** - Add proper error handling and fallbacks