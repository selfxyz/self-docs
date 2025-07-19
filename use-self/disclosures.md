---
icon: eye
---

# Disclosures

Disclosures control what information users reveal during identity verification. You configure them in the frontend `disclosures` object, which contains two types of settings:

1. **Verification Requirements** - conditions that must be met (must match backend)
2. **Disclosure Requests** - information users will reveal (frontend only)

## Verification Requirements

These settings define verification conditions and must match your backend `verification_config`:

### `minimumAge`
Verifies user is at least this age without revealing exact age or date of birth.

<<<<<<< HEAD
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
=======
```javascript
disclosures: {
  minimumAge: 18, // User must be 18 or older
>>>>>>> eff2f79 (Revise disclosures documentation to clarify the purpose and configuration of verification requirements and disclosure requests. Introduce new sections for `minimumAge`, `excludedCountries`, and `ofac`, along with examples for better understanding. Emphasize privacy best practices and common use cases for disclosures.)
}
```

### `excludedCountries` 
Blocks users from specific countries using ISO 3-letter country codes.

```javascript
disclosures: {
  excludedCountries: ['IRN', 'PRK'], // Block Iran and North Korea
}
```

### `ofac`
Enables OFAC (sanctions) checking against official watchlists.

```javascript
disclosures: {
  ofac: true, // Enable sanctions checking
}
```

## Disclosure Requests

These settings specify what information users will reveal. Configure only in frontend - backend receives this data automatically.

### Personal Information

- **`name`**: User's full name from passport
- **`nationality`**: User's nationality 
- **`gender`**: User's gender (M/F)
- **`date_of_birth`**: Full date of birth

### Document Information

- **`passport_number`**: Passport number (use carefully for privacy)
- **`expiry_date`**: Passport expiry date
- **`issuing_state`**: Country that issued the passport

## Example Configuration

```javascript
disclosures: {
  // Verification requirements (must match backend)
  minimumAge: 21,
  excludedCountries: ['IRN'],
  ofac: true,
  
  // Disclosure requests (frontend only)
  nationality: true,
  gender: true,
  name: false,           // Don't request name
  date_of_birth: true,
  passport_number: false, // Don't request for privacy
}
```

## Verification Result

When verification succeeds, disclosed information is available in `result.discloseOutput`:

```javascript
const result = await selfBackendVerifier.verify(/*...*/);
if (result.isValidDetails.isValid) {
  const data = result.discloseOutput;
  
  console.log(data.nationality);    // "USA" (if requested)
  console.log(data.gender);         // "M" or "F" (if requested)
  console.log(data.olderThan);      // "18" (if minimumAge set)
  console.log(data.name);           // undefined (if not requested)
}
```

## Privacy Best Practices

- **Request only what you need**: Each disclosure reveals personal information
- **Avoid sensitive fields**: Be cautious with `passport_number` and `name`
- **Consider alternatives**: Use `minimumAge` instead of `date_of_birth` for age verification
- **Store carefully**: Implement proper data protection for disclosed information

## Common Use Cases

**Age verification only:**
```javascript
disclosures: {
  minimumAge: 18, // No personal data revealed
}
```

**Basic identity with nationality:**
```javascript
disclosures: {
  minimumAge: 18,
  nationality: true,
  gender: true,
}
```

**Complete identity verification:**
```javascript
disclosures: {
  minimumAge: 21,
  ofac: true,
  nationality: true,
  name: true,
  date_of_birth: true,
}
```