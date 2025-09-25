---
description: API Reference for @selfxyz/qrcode
---

# QRCode SDK - API Reference

## Exports

* SelfQRCodeWrapper
* SelfQRCode
* SelfAppBuilder
* SelfApp
* getUniversalLink

## SelfQRCodeWrapper / SelfQRCode

| Input        | Type                                                      | Default         | Description                                          |
| ------------ | --------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| selfApp      | [#selfapp](qrcode-sdk-api-reference.md#selfapp "mention") | -               | The configured Self app instance.                    |
| onSuccess    | () ⇒ void;                                                | -               | Callback triggered when verification succeeds.       |
| onError      | data: { error\_code?: string; reason?: string }) => void  | -               | Callback triggered when verification fails.          |
| type         | 'websocket' \| 'deeplink'                                 | websocket       | Determines whether to use WebSocket or deep link QR. |
| websocketUrl | string                                                    | WS\_DB\_RELAYER | Custom WebSocket relayer URL.                        |
| size         | number                                                    | 300             | Width and height of the QR Code in pixels.           |
| darkMode     | boolean                                                   | false           | Toggles light/dark mode for QR code styling.         |

## SelfApp

| Property         | Type                                                                                      | Required | Description                                                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------- |
| appName          | string                                                                                    | ✅        | The name of your app                                                                                                         |
| logoBase64       | string                                                                                    | ✅        | Image URL or base64 encoded image                                                                                            |
| endpointType     | `https` \| `stating_https` \| `celo` \| `staging_celo`                                    | ✅        | Required by the Self Protocol to know where the proofs will be verified: Onchain/Offchain and Real Documents/Mock Documents. |
| endpoint         | string                                                                                    | ✅        | Either the EVM Address or the backend URL where the proof must be verified.                                                  |
| deeplinkCallback | string                                                                                    | ⚪        | Triggered by the app after proof the proof is verified (or if it fails being generated)                                      |
| scope            | string                                                                                    | ✅        | A unique identifier for you application                                                                                      |
| userId           | string                                                                                    | ✅        | An identifier for the end user.                                                                                              |
| userIdType       | 'uuid' \| 'hex'                                                                           | ✅        | Type of the user identifier                                                                                                  |
| disclosures      | [#selfappdisclosureconfig](qrcode-sdk-api-reference.md#selfappdisclosureconfig "mention") | ✅        | Object containing all disclosures and checks.                                                                                |
| version          | 1 \| 2                                                                                    | ⚪        | Whether to use Self V1 or SelfV2. Defaults to 2                                                                              |
| userDefinedData  | string                                                                                    | ⚪        | Any data you want to pass to your endpoint.                                                                                  |

## SelfAppDisclosureConfig

| Property              | Type                  | Default | Description                                               |
| --------------------- | --------------------- | ------- | --------------------------------------------------------- |
| issuing\_state        | boolean               | false   | Request the issuing state from the document               |
| name                  | boolean               | false   | Request the full name from the document                   |
| passport\_number      | boolean               | false   | Request the document number.                              |
| nationality           | boolean               | false   | Request the user’s nationality.                           |
| date\_of\_birth       | boolean               | false   | Request the date of birth.                                |
| gender                | boolean               | false   | Request the gender field.                                 |
| expiry\_date          | boolean               | false   | Request the passport expiry date.                         |
| ofac\*\*              | boolean               | false   | Check against OFAC sanction lists.                        |
| excludedCountries\*\* | Country3LetterCode\[] | \[]     | Exclude users from specific ISO 3166-1 alpha-3 countries. |
| minimumAge\*\*        | number                | 0       | Require a minimum age (e.g., `18` (upto `99`)).\|         |

