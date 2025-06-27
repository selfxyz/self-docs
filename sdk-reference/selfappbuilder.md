# SelfAppBuilder

A React component for generating QR codes for Self passport verification.

### V2 Updates

> 🚀 **New in V2**: The builder now requires `version` and `userDefinedData` parameters for enhanced functionality:
>
> * `version: 2` - Specifies the protocol version
> * `userDefinedData` - Custom data passed to your backend for context-aware verification

### Constructor

```typescript
new SelfAppBuilder(config: Partial<SelfApp>)
```

The `SelfAppBuilder` allows you to configure your application's verification requirements:



#### Parameters

| Parameter         | Type                                                           | Required | Description                                                           |
| ----------------- | -------------------------------------------------------------- | -------- | --------------------------------------------------------------------- |
| `appName`         | string                                                         | Yes      | The name of your application                                          |
| `scope`           | string                                                         | Yes      | A unique identifier for your application (max 25 chars)               |
| `endpoint`        | string                                                         | Yes      | The endpoint that will verify the proof                               |
| `endpointType`    | `'celo'` \| `'https'` \| `'staging_celo'` \| `'staging_https'` | No       | Whether the endpoint verifies proofs on-chain or off-chain            |
| `logoBase64`      | string                                                         | No       | Base64-encoded logo or PNG URL to display in the Self app             |
| `userId`          | string                                                         | Yes      | Unique identifier for the user                                        |
| `userIdType`      | `'uuid'` \| `'hex'`                                            | No       | `'hex'` for on-chain addresses, `'uuid'` for off-chain identification |
| `version`         | number                                                         | Yes (V2) | Protocol version (use `2` for latest)                                 |
| `userDefinedData` | string                                                         | Yes (V2) | 64-byte hex string for custom data                                    |
| `disclosures`     | object                                                         | No       | Disclosure and verification requirements                              |
| `devMode`         | boolean                                                        | No       | Enable development mode for testing                                   |

### Disclosure Options



The `disclosures` object can include the following options:

| Option              | Type      | Description                                    |
| ------------------- | --------- | ---------------------------------------------- |
| `issuing_state`     | boolean   | Request disclosure of passport issuing state   |
| `name`              | boolean   | Request disclosure of the user's name          |
| `nationality`       | boolean   | Request disclosure of nationality              |
| `date_of_birth`     | boolean   | Request disclosure of birth date               |
| `passport_number`   | boolean   | Request disclosure of passport number          |
| `gender`            | boolean   | Request disclosure of gender                   |
| `expiry_date`       | boolean   | Request disclosure of passport expiry date     |
| `minimumAge`        | number    | Verify the user is at least this age           |
| `excludedCountries` | string\[] | Array of ISO 3-letter country codes to exclude |
| `ofac`              | boolean   | Enable OFAC compliance check                   |

### Important Notes

#### Scope

* Must be unique to your application
* Maximum 25 characters
* Used to prevent cross-application proof replay
* Must match the scope configured in your backend verifier

#### User ID

* Use UUID for off-chain verification
* Use hex-encoded address for on-chain verification
* Ties the proof to a specific user

#### User Defined Data

* 64 bytes (128 hex characters) of custom data
* Passed to your backend's `IConfigStorage.getActionId()` method
* Use for context-aware verification configurations
* Examples: action types, transaction amounts, session IDs

#### Endpoint Requirements

* For `https` type: Must be publicly accessible (not localhost)
* For development: Use ngrok or similar tunneling service
* The Self backend relayer must be able to reach your endpoint

### Migration from V1

If you're upgrading from V1:

```javascript
// Old V1 pattern
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app",
  endpoint: "https://api.myapp.com/verify",
  userId: userId,
  disclosures: { /* ... */ }
}).build();

// New V2 pattern
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app",
  endpoint: "https://api.myapp.com/verify",
  userId: userId,
  version: 2,                    // Add version
  userDefinedData: "0x00...",    // Add user defined data
  disclosures: { /* ... */ }
}).build();
```
