# Hub Verification Process

The IdentityVerificationHub V2 provides enhanced verification for multiple document types with a structured framework.

## Verification Flow

1. **Document Detection**: Identifies E_PASSPORT or EU_ID_CARD proof type
2. **Configuration Retrieval**: Loads verification settings via configId
3. **Core Verification**: Validates scope, root, timestamp, and cryptographic proof
4. **Document-Specific Logic**: Applies type-specific verification rules
5. **Output Generation**: Returns structured identity data

### V2 Verifications Performed

In the IdentityVerificationHub V2, we execute these verifications:

* **Merkle Root Consistency:**\
  The proof must be generated using the Merkle tree maintained in our registry.
* **Timestamp Verification:**\
  The proof must have been generated recently, within a valid time window.
* **Minimum Age Check:**\
  If enabled, the proof must contain a valid age verification:
  * For example, if the contract is set to require an age of 18, then a proof generated for a user who is 20 will pass. Conversely, if the contract requires a minimum of 20 and the user’s proof indicates 18, the verification will fail.
* **Excluded Countries Check:**\
  If enabled, you can specify forbidden nationalities.
  * The forbidden countries are represented as a packed list of three-letter country codes.
  * For gas efficiency, the check is performed on the packed data without unpacking.
  * Ensure that the order of the forbidden countries specified in your integrated application matches the `forbiddenCountriesListPacked` value in the contract.
* **OFAC Check:**\
  Self supports OFAC checks against the U.S. Treasury’s sanctions list.
  * There are multiple OFAC verification methods:
    * `ofacEnabled[0]`: OFAC check using passport number.
    * `ofacEnabled[1]`: OFAC check using name and date of birth.
    * `ofacEnabled[2]`: OFAC check using name and year of birth.
  * Configure your contract to enable the same OFAC checks as those enabled in your application.
* **Enabled Flags:**\
  The flags (`olderThanEnabled`, `forbiddenCountriesEnabled`, and `ofacEnabled`) allow you to save gas by skipping unnecessary verifications.
* **Groth16 Proof Verification:**\
  Validates the cryptographic proof using the appropriate circuit verifier.

### V2 Key Improvements

V2 introduces significant architectural enhancements:

* **Multi-Document Support:**\
  Automatically routes verification based on document type (E_PASSPORT or EU_ID_CARD).
* **Structured Output:**\
  Returns rich `GenericDiscloseOutputV2` with pre-extracted attributes instead of raw field elements.
* **Flexible Configuration:**\
  Uses `VerificationConfigV2` stored in the hub with configId-based references, enabling reusable configurations and new configuration creation without contract redeployment. Use the [Self Configuration Tools](https://tools-mu-one.vercel.app/) to create and manage your verification configurations.

### V2 Output Structure

V2 verification returns a rich `GenericDiscloseOutputV2` structure containing:

```solidity
struct GenericDiscloseOutputV2 {
    bytes32 attestationId;           // E_PASSPORT (bytes32(uint256(1))) or EU_ID_CARD (bytes32(uint256(2)))
    uint256 userIdentifier;         // User's unique identifier
    uint256 nullifier;              // Anti-double-spending nullifier
    uint256[4] forbiddenCountriesListPacked; // Packed forbidden countries list
    string issuingState;            // Document issuing state/country
    string[] name;                  // Array of name components (first, middle, last)
    string idNumber;                // Document number (passport/ID card number)
    string nationality;             // User's nationality
    string dateOfBirth;            // Date of birth (format: "DD-MM-YY")
    string gender;                 // User's gender
    string expiryDate;            // Document expiration date
    uint256 olderThan;            // Verified minimum age
    bool[3] ofac;                 // OFAC verification results [passportNo, nameAndDob, nameAndYob]
}
```
