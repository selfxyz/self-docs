# Self Mobile SDK

The Self Mobile SDK enables React Native applications to integrate identity verification using passport and ID document scanning, NFC reading, and zero-knowledge proof generation.

## Installation

```bash
npm install @selfxyz/mobile-sdk-alpha
```

## Core Concepts

The Self Mobile SDK consists of four main components:

### 1. SelfClientProvider
The root provider component that wraps your app and provides SDK context to all child components. It requires configuration with adapters and event listeners.

### 2. Adapters
Platform-specific implementations that the SDK uses to interact with your app's environment:
- **Required**: `scanner`, `crypto`, `network`, `auth`, `documents`
- **Optional**: `analytics`, `logger`, `storage`

### 3. Event Listeners
Handlers for SDK events that manage navigation and user feedback during identity verification flows.

### 4. Hooks & Components
- `useSelfClient()` hook provides access to SDK methods in your components
- Optional UI components assist with document scanning instructions

## Architecture Overview

```
Your React Native App
├── SelfClientProvider (setup.md)
│   ├── Adapters (adapters.md)
│   │   ├── Scanner (native modules)
│   │   ├── Crypto (hashing, signing)
│   │   ├── Network (HTTP, WebSocket)
│   │   ├── Auth (private keys)
│   │   └── Documents (secure storage)
│   └── Event Listeners
│       ├── Navigation events
│       └── Progress/error events
└── Your Components
    └── useSelfClient() hook
``` 


## Identity Verification Flow

The SDK supports a complete identity verification workflow:

### 1. Document Onboarding (Required)
1. **MRZ Scanning** - Scan the machine-readable zone with camera to extract document details
2. **NFC Reading** - Use device NFC to read passport chip data (requires MRZ data as "password")
3. **Proof Generation** - Generate zero-knowledge proofs for identity verification

### 2. Identity Disclosure (Optional)
When apps request identity information:
1. **Consent Screen** - Show what data will be revealed and to whom
2. **Proof Generation** - Create selective disclosure proofs
3. **Progress Feedback** - Keep users informed during proof generation

### 3. Document Management (Optional)
- Support for multiple document types
- Document registration status tracking
- Secure document storage and retrieval

## UX Considerations

### MRZ Scanning Screen
- Clearly communicate this scans text, not a photo
- Highlight the MRZ (machine-readable zone) area
- Provide visual guides for proper document positioning

### NFC Reading Screen  
- NFC chip location varies by country and passport type
- Provide clear instructions and patience guidance
- Show progress during the scanning process

## Next Steps

1. **[Setup Guide](setup.md)** - Configure the SelfClientProvider
2. **[Adapters Guide](adapters.md)** - Implement required platform adapters  
3. **API Reference** - Use SDK methods in your components 


