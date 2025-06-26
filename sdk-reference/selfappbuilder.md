# SelfAppBuilder

A React component for generating QR codes for Self passport verification.



The `SelfAppBuilder` allows you to configure your application's verification requirements:

| Parameter      | Type                                                   | Required | Description                                                                    |
| -------------- | ------------------------------------------------------ | -------- | ------------------------------------------------------------------------------ |
| `appName`      | string                                                 | Yes      | The name of your application                                                   |
| `scope`        | string                                                 | Yes      | A unique identifier for your application                                       |
| `endpoint`     | string                                                 | Yes      | The endpoint that will verify the proof                                        |
| `endpointType` | `celo` \| `https` \| `staging_celo` \| `staging_https` | Yes      | Whether the endpoint verifies the proofs on chain or off chain                 |
| `logoBase64`   | string                                                 | No       | Base64-encoded logo to display in the Self app                                 |
| `userId`       | string                                                 | Yes      | Unique identifier for the user                                                 |
| `userIdType`   | `UserIdType`                                           | Yes      | Hex implies onchain address whereas uuid implies uuid identification of users. |
| `disclosures`  | object                                                 | No       | Disclosure and verification requirements                                       |

### Disclosure Options



The `disclosures` object can include the following options:

| Option              | Type      | Description                                  |
| ------------------- | --------- | -------------------------------------------- |
| `issuing_state`     | boolean   | Request disclosure of passport issuing state |
| `name`              | boolean   | Request disclosure of the user's name        |
| `nationality`       | boolean   | Request disclosure of nationality            |
| `date_of_birth`     | boolean   | Request disclosure of birth date             |
| `passport_number`   | boolean   | Request disclosure of passport number        |
| `gender`            | boolean   | Request disclosure of gender                 |
| `expiry_date`       | boolean   | Request disclosure of passport expiry date   |
| `minimumAge`        | number    | Verify the user is at least this age         |
| `excludedCountries` | string\[] | Array of country codes to exclude            |
| `ofac`              | boolean   | Enable OFAC compliance check                 |
