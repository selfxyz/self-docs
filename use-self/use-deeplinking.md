---
description: Generate deep links to open the Self mobile app directly
icon: mobile
---

# Use Deeplinking

Deep links allow users to open the Self mobile app directly instead of scanning QR codes. This provides a better mobile experience by eliminating the need to scan codes on the same device.

## When to Use Deep Links

- **Mobile web applications**: Users can tap a button to open Self app
- **Same-device verification**: Avoid QR code scanning on mobile
- **Messaging platforms**: Share verification links via SMS, email, or chat
- **Native mobile apps**: Direct integration with mobile applications

## Basic Usage

Generate a deep link from your SelfApp configuration:

```javascript
import { getUniversalLink } from '@selfxyz/core';
import { SelfAppBuilder } from '@selfxyz/qrcode';

// Create your SelfApp (same as for QR codes)
const selfApp = new SelfAppBuilder({
  // ... your configuration
}).build();

// Generate the deep link
const deeplink = getUniversalLink(selfApp);

// Use the deep link
window.open(deeplink, '_blank'); // Opens Self app
```

## Implementation Examples

### Simple Button

```javascript
function OpenSelfButton() {
  const handleOpenSelf = () => {
    const deeplink = getUniversalLink(selfApp);
    window.open(deeplink, '_blank');
  };

  return (
    <button onClick={handleOpenSelf}>
      Open Self App
    </button>
  );
}
```

### Mobile-First Experience

```javascript
function VerificationOptions() {
  const [deeplink, setDeeplink] = useState('');
  
  useEffect(() => {
    if (selfApp) {
      setDeeplink(getUniversalLink(selfApp));
    }
  }, [selfApp]);

  return (
    <div>
      {/* Show deep link button on mobile */}
      <div className="md:hidden">
        <button onClick={() => window.open(deeplink, '_blank')}>
          Open Self App
        </button>
      </div>
      
      {/* Show QR code on desktop */}
      <div className="hidden md:block">
        <SelfQRcodeWrapper selfApp={selfApp} />
      </div>
    </div>
  );
}
```

## Platform Considerations

### Mobile Browsers
- **iOS Safari**: Deep links work reliably
- **Android Chrome**: Deep links work reliably  
- **In-app browsers**: May have limitations

### Desktop Browsers
- Deep links will attempt to open the mobile app
- If app not installed, may show app store page
- Generally better to show QR codes on desktop


## Best Practices

- **Provide both options**: Offer both QR codes and deep links
- **Mobile-first**: Prioritize deep links on mobile devices
- **Clear labeling**: Make it obvious what the button does
- **Testing**: Test on various mobile browsers and devices