---
hidden: true
---

# Self Attestation

The `SelfAttestation` interface represents the structure of an attestation used in the Self system. It includes details about the credential subject, proof, and document signing certificate (DSC) proof. It has been designed according to W3C standards.

#### Parameters

| Parameter           | Type       | Description                                      |
| ------------------- | ---------- | ------------------------------------------------ |
| `@context`          | `string[]` | Context URLs for the attestation.                |
| `type`              | `string[]` | Types of the attestation.                        |
| `issuer`            | `string`   | The issuer of the attestation.                   |
| `issuanceDate`      | `string`   | The date the attestation was issued.             |
| `credentialSubject` | `object`   | Contains details about the user and application. |
| `proof`             | `object`   | Contains passport proof.                         |
| `dscProof`          | `object`   | Contains DSC proof details for verification.     |
| `dsc`               | `object`   | Contains DSC details.                            |

### `OpenPassportDynamicAttestation`

The `OpenPassportDynamicAttestation` class extends `OpenPassportAttestation` and provides methods to parse and retrieve specific attributes from the attestation.

#### Functions

<table><thead><tr><th width="800">Function</th><th width="126">Parameters</th><th width="309">Description</th><th>Output</th></tr></thead><tbody><tr><td><code>getUserId</code></td><td>None</td><td>Retrieves the user ID from the parsed public signals.</td><td><code>string</code></td></tr><tr><td><code>getNullifier</code></td><td>None</td><td>Retrieves the nullifier from the parsed public signals.</td><td><code>string</code></td></tr><tr><td><code>getCommitment</code></td><td>None</td><td>Retrieves the commitment from the parsed public signals.</td><td><code>string</code></td></tr><tr><td><code>getNationality</code></td><td>None</td><td>Retrieves the nationality from the unpacked reveal data.</td><td><code>string</code></td></tr><tr><td><code>getCSCAMerkleRoot</code></td><td>None</td><td>Retrieves the CSCAMerkleRoot from the DSC proof public signals.</td><td><code>string</code></td></tr></tbody></table>
