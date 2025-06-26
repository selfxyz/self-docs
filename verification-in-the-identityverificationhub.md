# Verification in the IdentityVerificationHub

IdentityVerificationHub is the verification contract deployed by us. In the IdentityVerificationHub, we are executing these verifications.

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
*   **Verification groth16 proof itself**

    Call right groth16 verifier.
