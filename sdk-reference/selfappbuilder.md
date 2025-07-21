# SelfAppBuilder

A builder class for creating Self application configuration objects that define identity verification requirements.

## Constructor

```typescript
new SelfAppBuilder(config: Partial<SelfApp>)
```

The `SelfAppBuilder` creates and validates configuration objects that specify verification requirements for Self identity proofs.

### Parameters

| Parameter         | Type                                                           | Required | Description                                                           |
| ----------------- | -------------------------------------------------------------- | -------- | --------------------------------------------------------------------- |
| `appName`         | string                                                         | Yes      | The name of your application (shown in Self mobile app)              |
| `scope`           | string                                                         | Yes      | A unique identifier for your application (max 30 chars)               |
| `endpoint`        | string                                                         | Yes      | The endpoint that will verify the proof                               |
| `endpointType`    | `'celo'` \| `'https'` \| `'staging_celo'` \| `'staging_https'` | No       | Whether the endpoint verifies proofs on-chain or off-chain            |
| `logoBase64`      | string                                                         | No       | Base64-encoded logo or PNG URL to display in the Self app             |
| `userId`          | string                                                         | Yes      | Unique identifier for the user                                        |
| `userIdType`      | `'uuid'` \| `'hex'`                                            | No       | `'hex'` for on-chain addresses, `'uuid'` for off-chain identification |
| `version`         | number                                                         | No       | Protocol version (defaults to `2`)                                    |
| `userDefinedData` | string                                                         | No       | 64-byte hex string for custom data (defaults to empty)                |
| `disclosures`     | object                                                         | No       | Disclosure and verification requirements                              |
| `devMode`         | boolean                                                        | No       | Enable development mode for testing                                   |

### Verification Requirements (disclosures)

The `disclosures` object defines what information to request and verify:

| Option              | Type      | Description                                    | Use Case                                     |
| ------------------- | --------- | ---------------------------------------------- | -------------------------------------------- |
| `issuing_state`     | boolean   | Request disclosure of document issuing state   | Country-specific services                    |
| `name`              | boolean   | Request disclosure of the user's name          | KYC, personalized services                   |
| `nationality`       | boolean   | Request disclosure of nationality              | Compliance, eligibility checks               |
| `date_of_birth`     | boolean   | Request disclosure of birth date               | Age verification, compliance                 |
| `passport_number`   | boolean   | Request disclosure of document number          | Unique identification                        |
| `gender`            | boolean   | Request disclosure of gender                   | Demographic analysis                         |
| `expiry_date`       | boolean   | Request disclosure of document expiry date     | Document validity                            |
| `minimumAge`        | number    | Verify the user is at least this age           | Age-gated services (18+, 21+, etc.)         |
| `excludedCountries` | string\[] | Array of ISO 3-letter country codes to exclude | Sanctions compliance, regional restrictions  |
| `ofac`              | boolean   | Enable OFAC compliance check                   | Financial services, sanctions screening      |

## Methods

### build()

Returns the configured `SelfApp` object.

w```typescript
build(): SelfApp
```