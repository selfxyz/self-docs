---
description: >-
  This guide shows how to drop in the SelfQRcode component from @selfxyz/qrcode
  to verify a user’s identity from a desktop browser (or a phone!) using the
  Self mobile app.
---

# QRCode SDK

> **Scope**: How to install, configure, and embed the component. [qrcode-sdk-api-reference.md](qrcode-sdk-api-reference.md "mention") (all props, types) lives on a separate page.

## Installation&#x20;

{% tabs %}
{% tab title="npm" %}
npm i @selfxyz/qrcode
{% endtab %}

{% tab title="yarn " %}
yarn add @selfxyz/core
{% endtab %}

{% tab title="pnpm " %}
pnpm i @selfxyz/qrcode
{% endtab %}
{% endtabs %}

## Usage (Desktop)

The `SelfQRCode` component is responsible for storing information about your app. This information includes data about the mock / real passports, the userId you want to store etc. along with information about the config that should be checked for a given proof.&#x20;

This involves 2 steps:&#x20;

1. Create a `SelfApp` from the `SelfAppBuilder` that takes in `Partial<SelfApp>`.
2. Pass the `SelfApp` to the QRCode component along with `onSuccess` and `onError` functions.&#x20;

{% code fullWidth="false" %}
```tsx
import { useEffect, useState } from 'react'
import { countries, SelfQRcodeWrapper } from '@selfxyz/qrcode'
import { SelfAppBuilder } from '@selfxyz/qrcode'

export default function Verify() {
  const [selfApp, setSelfApp] = useState<any | null>(null)

  useEffect(() => {
    const userId = '0xYourUserEthAddress' // or a UUID depending on your setup
    
    const app = new SelfAppBuilder({
      version: 2,
      appName: process.env.NEXT_PUBLIC_SELF_APP_NAME || 'Self Docs',
      scope: process.env.NEXT_PUBLIC_SELF_SCOPE || 'self-docs',
      endpoint: `${process.env.NEXT_PUBLIC_SELF_ENDPOINT}`,
      logoBase64: 'https://i.postimg.cc/mrmVf9hm/self.png',
      userId,
      endpointType: 'staging_celo',
      userIdType: 'hex', // 'hex' for EVM address or 'uuid' for uuidv4
      userDefinedData: 'Hello from the Docs!!',
      disclosures: {
        // What you want to verify from the user's identity
        minimumAge: 18,
        excludedCountries: [countries.CUBA, countries.IRAN, countries.NORTH_KOREA, countries.RUSSIA],

        // What you want users to
        nationality: true,
        gender: true,
      },
    }).build()

    setSelfApp(app)
  }, [])

  const handleSuccessfulVerification = () => {
    // Persist the attestation / session result to your backend, then gate content
    console.log('Verified!')
  }

  return (
    <div>
      {selfApp ? (
        <SelfQRcodeWrapper
          selfApp={selfApp}
          onSuccess={handleSuccessfulVerification}
          onError={() => {
            console.error('Error: Failed to verify identity')
          }}
        />
      ) : (
        <div>
          <p>Loading QR Code...</p>
        </div>
      )}
    </div>
  )
}
```
{% endcode %}

{% hint style="warning" %}
The claims you want to verify MUST exactly match the ones you set in your backend configuration.
{% endhint %}

{% hint style="danger" %}
If you're using a contract to verify your proofs then please sure the contract address is in lowercase.&#x20;
{% endhint %}

## Usage (Mobile)

If you're developing an app then it's not easy to scan a QR Code. Instead, what you would do is to create a deeplink to the Self app and pass in a `deeplinkCallback` url that the Self app can navigate to once the proof is verified. Here's what you'll need to add:&#x20;

```tsx
import { useEffect, useState } from 'react'
import { countries, SelfQRcodeWrapper } from '@selfxyz/qrcode'
import { SelfAppBuilder } from '@selfxyz/qrcode'

export default function Verify() {
  const [selfApp, setSelfApp] = useState<any | null>(null);
  const [universalLink, setUniversalLink] = useState("");

  useEffect(() => {
    const userId = '0xYourUserEthAddress' // or a UUID depending on your setup
    
    const app = new SelfAppBuilder({
      ..., //same as previous example 
      deeplinkCallback: "https://your-callback-url.com", 
    }).build()

    setSelfApp(app); 
    setUniversalLink(getUniversalLink(app);
  }, []);

  const openSelfApp = () => {
    if (!universalLink) return;
    window.open(universalLink, "_blank");
  }

  return (
    <div>
      {selfApp ? (
         <button
            type="button"
            onClick={openSelfApp}
            disabled={!universalLink}
          >
          Open Self App
        </button>
      ) : (
        <div>
          <p>Loading QR Code...</p>
        </div>
      )}
    </div>
  )
}
```

&#x20;
