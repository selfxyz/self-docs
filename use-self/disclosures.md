---
icon: eye
---

# Disclosures

Disclosures allow you to reveal information about your passport. For example, if you want to check if a user is above the age of 18 then at the very least you will end up disclosing the lower bound of the age range of the user.

### How to set the minimum age

In your QRCode component, you must first make sure that you disclose the minimum age before sending it to our servers to generate the proof.&#x20;

```typescript
import { SelfAppBuilder } from '@selfxyz/qrcode';
import { v4 as uuidv4 } from 'uuid';

const userId = uuidv4();
​
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app-scope", 
  endpoint: "https://myapp.com/api/verify",
  endpointType: "https",
  logoBase64: "<base64EncodedLogo>",
  userId,
  disclosures: {
    minimumAge: 18,
  }
}).build();
```

Once you've made the changes in the frontend, you can change your backend verifier to make use of  the `SelfBackendVerifier` ,checking for the minimum age is as simple as:&#x20;

```typescript
selfBackendVerifier.setMinimumAge(18);
```

Note that for all types of disclosures, the frontend and backend have to be consistent. For example, if you create a proof from the frontend to disclose a minimum age of 17 and configure the backend to accept proofs with a minimum age of 16, verification fails.

Take a look at [selfbackendverifier.md](../sdk-reference/selfbackendverifier.md "mention") for the different types of methods you can use to configure the `SelfBackendVerifier`.

If you want to add your own custom checks for verifying users, then you can use the `verify` function to get all the disclosed attributes of the proof. The result of the `verify` function is.

```typescript
export interface SelfVerificationResult {
  // Check if the whole verification has succeeded
  isValid: boolean;
  isValidDetails: {
    // Verifies that the proof is generated under the expected scope.
    isValidScope: boolean;
    //Check that the proof's attestation identifier matches the expected value.
    isValidAttestationId: boolean;
    // Verifies the cryptographic validity of the proof.
    isValidProof: boolean;
    // Ensures that the revealed nationality is correct (when nationality verification is enabled).
    isValidNationality: boolean;
  };
  // User Identifier which is included in the proof
  userId: string;
  // Application name, which is the same as scope
  application: string;
  // A cryptographic value used to prevent double registration or reuse of the same proof.
  nullifier: string;
  // Revealed data by users
  credentialSubject: {
    // Merkle root, which is used to generate proof.
    merkle_root?: string;
    // Proved identity type. For passport, this value is fixed as 1.
    attestation_id?: string;
    // Date when the proof is generated
    current_date?: string;
    // Revealed issuing state in the passport
    issuing_state?: string;
    // Revealed name in the passport
    name?: string;
    // Revealed passport number in the passport
    passport_number?: string;
    // Revealed nationality in the passport
    nationality?: string;
    // Revealed date of birth in the passport
    date_of_birth?: string;
    // Revealed gender in the passport
    gender?: string;
    // Revealed expiry date in the passport
    expiry_date?: string;
    // Result of older than
    older_than?: string;
    // Result of passport number ofac check
    // Gives true if the user passed the check (is not on the list),
    // false if the check was not requested or if the user is in the list
    passport_no_ofac?: boolean;
    // Result of name and date of birth ofac check
    name_and_dob_ofac?: boolean;
    // Result of name and year of birth ofac check
    name_and_yob_ofac?: boolean;
  };
  proof: {
    // Proof that is used for this verification
    value: {
      proof: Groth16Proof;
      publicSignals: PublicSignals;
    };
  };
```

The `credentialSubject` field includes all the disclosed attributes. If you did not configure to disclose a certain attribute, then the value of the field is going to be `undefined`.&#x20;

For example, assuming you disclosed the nationality of the user, then you can add a check in the backend verifier to check that they are not from North Korea.&#x20;

```typescript
import { countries, SelfVerificationResult } from "@selfxyz/core";

const result: SelfVerificationResult = await selfBackendVerifier.verify(request.body.proof, request.body.publicSignals);
if (result.credentialSubject.nationality === countries.IRAN ) { 
    return res.status(200).json({
          status: 'failure',
          result: false,
          reason: "User nationality is not allowed",
          error_code: "RESTRICTED_NATIONALITY"
    });
}
```

### Nullifiers

If you're verifying proofs offchain, you will want to store nullifiers. If someone tries to do two proofs with the same passport, you can reject them as you see their new proof has the same nullifier.

For onchain usage, this is handled by smart contracts. You can see an example in our [happy birthday contract](https://github.com/selfxyz/happy-birthday/blob/main/contracts/contracts/HappyBirthday.sol).
