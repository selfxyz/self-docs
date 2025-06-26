# SelfBackendVerifier

The `SelfBackendVerifier` class is designed to facilitate the verification of user credentials and disclosures in applications using the Self system. It supports various modes of operation, allowing for both onchain and offchain proof verification. The class provides methods to configure verification parameters such as minimum age, nationality, and OFAC checks, and to generate intents for user interactions.

#### Parameters

| Parameter              | Type         | Description                                                            |
| ---------------------- | ------------ | ---------------------------------------------------------------------- |
| `scope`                | `string`     | An identifier for the application, used to distinguish different apps. |
| `endpoint`             | `string`     | The endpoint of the backend verifier.                                  |
| `user_identifier_type` | `UserIdType` | The type of the user identifier. Hex denotes on chain addresses.       |
| `mockPassport`         | `boolean`    | The passport type - false if the backend is verifying real passports.  |

#### Functions

<table><thead><tr><th width="170.6015625">Function</th><th>Parameters</th><th>Description</th><th>Output</th></tr></thead><tbody><tr><td><code>setMinimumAge</code></td><td><code>age: number</code></td><td>Sets the minimum age requirement for verification. Throws an error if age is less than 10 or more than 100.</td><td><code>this</code></td></tr><tr><td><code>setNationality</code></td><td><code>country: (typeof countryNames)[number]</code></td><td>Sets the nationality requirement for verification.</td><td><code>this</code></td></tr><tr><td><code>excludeCountries</code></td><td><code>...countries: (typeof countryNames)[number][]</code></td><td>Excludes specified countries from verification.</td><td><code>this</code></td></tr><tr><td><code>enablePassportNoOfacCheck</code></td><td></td><td>Checks for the passport number in the OFAC list. </td><td><code>this</code></td></tr><tr><td><code>enableNameAndDobOfacCheck</code></td><td></td><td>Checks for the name and DOB (hashed together) in the OFAC list.</td><td><code>this</code></td></tr><tr><td><code>enableNameAndYobOfacCheck</code></td><td></td><td>Checks for the name and year of birth (hashed together) in the OFAC list. </td><td><code>this</code></td></tr><tr><td><code>verify</code></td><td><code>proof: OpenPassportAttestation</code></td><td>Verifies a proof against the configured verification parameters.</td><td><code>Promise&#x3C;OpenPassportVerifierReport></code></td></tr></tbody></table>
