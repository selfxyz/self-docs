# Onboarding Screen Components

The Mobile SDK provides pre-built React Native screen components for the identity verification onboarding flow. These screens handle the UI and logic for document selection, camera scanning, and user interactions.

## Import Pattern

Screens are imported directly from their onboarding paths, not from the main package index:

```tsx
import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';
import IDSelectionScreen from '@selfxyz/mobile-sdk-alpha/onboarding/id-selection-screen';  
import SDKCountryPickerScreen from '@selfxyz/mobile-sdk-alpha/onboarding/country-picker-screen';
```

## Provider Requirement

⚠️ **All screens must be wrapped in `SelfClientProvider`**. The screens use `useSelfClient()` internally and will throw an error if used outside the provider context.

## Available Screens

### DocumentCameraScreen

Handles MRZ (Machine Readable Zone) scanning from identity documents using the device camera.

```tsx
import { DocumentCameraScreen } from '@selfxyz/mobile-sdk-alpha/onboarding/document-camera-screen';

<DocumentCameraScreen 
  onBack={() => navigation.goBack()}
  onSuccess={() => navigation.navigate('NFCScan')}
  safeAreaInsets={{ top: 44, bottom: 34 }} // Optional
/>
```

**Props:**
- `onBack?: () => void` - Called when user taps back button
- `onSuccess?: () => void` - Called when MRZ scan succeeds  
- `safeAreaInsets?: { top: number, bottom: number }` - Safe area padding

**Behavior:**
- Displays camera viewfinder with MRZ scanning overlay
- Shows scan instructions and animations
- Automatically processes MRZ data when detected
- Emits `DOCUMENT_MRZ_READ_SUCCESS` or `DOCUMENT_MRZ_READ_FAILURE` events
- Provides haptic feedback during scanning

### IDSelectionScreen

Displays available document types for a selected country and handles document type selection.

```tsx
import IDSelectionScreen from '@selfxyz/mobile-sdk-alpha/onboarding/id-selection-screen';

<IDSelectionScreen 
  countryCode="USA"
  documentTypes={['p', 'i']} // p = passport, i = ID card, a = aadhaar
/>
```

**Props:**
- `countryCode: string` - ISO country code (e.g., "USA", "GBR") 
- `documentTypes: string[]` - Available document types for the country

**Behavior:**
- Shows document type cards with icons (passport, ID card, Aadhaar, etc.)
- Handles document type selection
- Emits `DOCUMENT_TYPE_SELECTED` event with selected type and country
- Provides visual feedback for selections

### CountryPickerScreen

Displays a searchable list of countries for document selection.

```tsx
import SDKCountryPickerScreen from '@selfxyz/mobile-sdk-alpha/onboarding/country-picker-screen';

<SDKCountryPickerScreen />
```

**Props:**
- No props required - manages its own state internally

**Behavior:**
- Shows searchable country list with flags
- Filters countries by name as user types
- Loads available document types for each country
- Emits `DOCUMENT_COUNTRY_SELECTED` event with country info
- Optimized for performance with large country lists

## Screen Flow Example

A typical onboarding flow might look like:

```
CountryPickerScreen 
    ↓ (DOCUMENT_COUNTRY_SELECTED)
IDSelectionScreen
    ↓ (DOCUMENT_TYPE_SELECTED) 
DocumentCameraScreen
    ↓ (DOCUMENT_MRZ_READ_SUCCESS)
[Your NFC Scanning Screen]
```

## Error Handling

Screens handle errors internally and emit appropriate events. Listen for failure events in your event listeners:

```tsx
addListener(SdkEvents.DOCUMENT_MRZ_READ_FAILURE, () => {
  // Navigate to error screen or retry flow
  navigation.navigate('DocumentScanError');
});
```

## Related Documentation

- [Getting Started Guide](getting-started.md) - SDK overview and installation
- [SelfClient Provider Setup](selfclient-provider.md) - Configure adapters and event listeners
- [Examples](examples/README.md) - Complete implementation examples