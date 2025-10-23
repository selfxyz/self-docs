# Navigation Integration Example

This example shows how to properly set up event listeners to create a seamless navigation flow between SDK screens.

## Event Listener Setup

The SDK uses an event-driven architecture. Set up listeners in your `SelfClientProvider` to handle navigation between screens.

```tsx
import { createListenersMap, SdkEvents } from '@selfxyz/mobile-sdk-alpha';

function createNavigationListeners(navigation) {
  const { map, addListener } = createListenersMap();

  // Country selection -> ID picker
  addListener(SdkEvents.DOCUMENT_COUNTRY_SELECTED, ({ countryCode, documentTypes }) => {
    navigation.navigate('IDPicker', { countryCode, documentTypes });
  });

  // Document type selection -> appropriate flow
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
        navigation.navigate('ComingSoon', { documentType, countryCode });
    }
  });

  // MRZ scan success -> NFC scanning
  addListener(SdkEvents.DOCUMENT_MRZ_READ_SUCCESS, () => {
    navigation.navigate('NFCScan');
  });

  // MRZ scan failure -> troubleshooting
  addListener(SdkEvents.DOCUMENT_MRZ_READ_FAILURE, () => {
    navigation.navigate('DocumentTrouble');
  });

  return map;
}
```

## Integration with Navigation

```tsx
function App() {
  const navigation = useNavigation();
  const listeners = useMemo(() => createNavigationListeners(navigation), [navigation]);
  
  return (
    <SelfClientProvider 
      config={config}
      adapters={adapters} 
      listeners={listeners}
    >
      <Stack.Navigator>
        <Stack.Screen name="CountryPicker" component={CountryPickerScreen} />
        <Stack.Screen name="IDPicker" component={IDPickerScreen} />  
        <Stack.Screen name="DocumentCamera" component={DocumentCameraScreen} />
      </Stack.Navigator>
    </SelfClientProvider>
  );
}
```

## Screen Components

```tsx
function CountryPickerScreen() {
  return <SDKCountryPickerScreen />;
}

function IDPickerScreen({ route }) {
  const { countryCode, documentTypes } = route.params;
  return <IDSelectionScreen countryCode={countryCode} documentTypes={documentTypes} />;
}

function DocumentCameraScreen({ navigation }) {
  return (
    <DocumentCameraScreen
      onBack={() => navigation.goBack()}
      onSuccess={() => console.log('Scan successful')}
    />
  );
}
```

## Event Flow

```
CountryPickerScreen 
    ↓ (DOCUMENT_COUNTRY_SELECTED)
IDSelectionScreen
    ↓ (DOCUMENT_TYPE_SELECTED) 
DocumentCameraScreen
    ↓ (DOCUMENT_MRZ_READ_SUCCESS)
[Your NFC Screen]
```

## Related Documentation

- [Getting Started Guide](../getting-started.md) - SDK overview and installation
- [SelfClient Provider Setup](../selfclient-provider.md) - Detailed adapter and event listener configuration
- [Minimal Setup](minimal-setup.md) - Basic integration example
- [Demo Walkthrough](demo-walkthrough.md) - Production implementation patterns