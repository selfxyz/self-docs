# Architecture & How It Works

## Self Connect: Architecture & How It Works

### Protocol Architecture

Self Connect uses a federated architecture where multiple independent issuers can register attestations that map obfuscated identifiers to blockchain addresses.

#### Core Components

```
┌─────────────────┐
│   User/Client   │
└────────┬────────┘
         │
         ├──────────────┐
         │              │
         v              v
┌─────────────┐  ┌─────────────┐
│    ODIS     │  │   Issuer    │
│  (Privacy)  │  │(Verification)│
└──────┬──────┘  └──────┬──────┘
       │                │
       └────────┬───────┘
                │
                v
     ┌──────────────────────┐
     │ FederatedAttestations│
     │   Smart Contract     │
     └──────────────────────┘
```

#### FederatedAttestations Smart Contract

The `FederatedAttestations` contract is the on-chain registry for all attestation mappings. It stores attestations in a structure that allows:

* **Registration:** Issuers register mappings between obfuscated identifiers and addresses
* **Lookup:** Anyone can query attestations for a given obfuscated identifier
* **Multi-Issuer Support:** Attestations are organized by issuer, allowing trust-based filtering
* **Batch Operations:** Multiple attestations can be queried simultaneously

**Key Functions:**

```solidity
// Register an attestation as an issuer
function registerAttestationAsIssuer(
    bytes32 identifier,
    address account,
    uint64 issuedOn
) external

// Lookup attestations for an identifier across multiple issuers
function lookupAttestations(
    bytes32 identifier,
    address[] calldata trustedIssuers
) external view returns (
    uint256[] memory countsPerIssuer,
    address[] memory accounts,
    address[] memory signers,
    uint64[] memory issuedOns,
    uint64[] memory publishedOns
)
```

**Contract Address:**

* Mainnet: [View on Celoscan](https://celoscan.io/)
* Alfajores Testnet: [View on Celoscan](https://alfajores.celoscan.io/)

#### Issuer Ecosystem

Issuers are independent entities that:

1. **Verify** user ownership of identifiers through their chosen method
2. **Request** obfuscated identifiers from ODIS
3. **Register** attestations to the FederatedAttestations contract
4. **Maintain** ODIS quota for continued operations

Applications choose which issuers to trust based on:

* Verification rigor
* Reputation
* Use case alignment
* Uptime and reliability

### ODIS (Oblivious Decentralized Identifier Service)

ODIS is the privacy-preserving component of Self Connect that ensures identifiers cannot be reverse-engineered from on-chain data.

#### What is ODIS

ODIS implements a rate-limited Oblivious Pseudorandom Function (OPRF) that allows users to compute a limited number of hashes without letting the service see the data being hashed.

**Key Properties:**

* **Oblivious:** ODIS operators cannot see the plaintext identifiers
* **Decentralized:** No single party can compute the pepper alone
* **Rate-Limited:** Prevents rainbow table attacks through quota system
* **Deterministic:** Same input always produces same output

#### Privacy Guarantees

ODIS provides strong privacy guarantees through its design:

**1. Blinding Process**

When a client queries ODIS:

```
1. Client blinds the identifier locally using a secret one-time key
2. Blinded value is sent to ODIS operators
3. Operators compute the OPRF on the hidden input
4. Operators return the blinded result
5. Client unblinds the result to get the final pepper
```

This process ensures that:

* ODIS operators never see the plaintext identifier
* Even if all operators are compromised, user privacy is maintained
* No targeted censorship is possible

**2. Threshold Cryptography**

ODIS uses a (k, m) threshold signature scheme:

* **m:** Total number of ODIS operators
* **k:** Minimum number of signatures required

**Security Properties:**

* If fewer than k operators are compromised: Attackers cannot compute unauthorized peppers
* If at least k operators are honest: Service remains available for legitimate users

**Example Configuration:**

* 7 operators (m=7)
* 5 required signatures (k=5)
* Security: Need 5 compromised operators to break privacy
* Availability: Can tolerate 2 operators being offline

**3. Distributed Key Generation (DKG)**

Before deployment, ODIS operators participated in a DKG ceremony to generate a shared secret, split across all operators. Each operator holds a key share that can be used to sign responses.

When enough signatures (≥k) are combined, they produce the unique, deterministic pepper for that identifier.

#### Quota System and Rate Limiting

ODIS implements a quota system to prevent rainbow table attacks while allowing legitimate usage.

**Quota Factors**

Quota is based on:

1. **Payment:** Pay for quota using cUSD
   * 10 cUSD = 10,000 queries
   * \~0.001 cUSD per query
2. **Account-based Limits:** Rate limits per account to prevent abuse

**How Quota Works**

```typescript
// Check current quota
const { remainingQuota } = await OdisUtils.Quota.getPnpQuotaStatus(
  issuerAddress,
  authSigner,
  serviceContext
);

// Purchase quota if needed
if (remainingQuota < 1) {
  const stableTokenContract = await kit.contracts.getStableToken();
  const odisPaymentsContract = await kit.contracts.getOdisPayments();
  const ONE_CENT_CUSD_WEI = 10000000000000000;
  
  await stableTokenContract
    .increaseAllowance(odisPaymentsContract.address, ONE_CENT_CUSD_WEI)
    .sendAndWaitForReceipt();
    
  await odisPaymentsContract
    .payInCUSD(issuerAddress, ONE_CENT_CUSD_WEI)
    .sendAndWaitForReceipt();
}
```

**Rate Limiting Strategy**

The quota system makes it prohibitively expensive to:

* Scrape large quantities of identifiers
* Build rainbow tables
* Perform mass surveillance

While still allowing:

* Normal user flows
* Legitimate application usage
* Reasonable issuer operations

#### Key Rotation

If an operator's key is leaked or compromised, ODIS can perform key rotation:

1. New DKG ceremony with at least k old keys participating
2. New keys generated for all operators (including new ones)
3. Old keys destroyed after successful rotation
4. Public verification key remains unchanged

This allows:

* Adding new operators
* Removing compromised operators
* Changing threshold values (k, m)
* Maintaining service continuity

### Identifier Obfuscation Process

The obfuscation process transforms a plaintext identifier into a privacy-preserving on-chain identifier.

#### Step-by-Step Process

**1. Format the Identifier**

Combine the identifier prefix with the plaintext identifier:

```
formatted = "{prefix}://{plaintextIdentifier}"
```

**Example:**

```
Phone: "PHONE_NUMBER://+12345678901"
Twitter: "twitter://@alice"
```

**2. Hash the Formatted Identifier**

```
hashedIdentifier = sha3(formatted)
```

**3. Blind the Hash**

The client generates a random blinding factor and blinds the hash:

```
blindedHash = blind(hashedIdentifier, blindingFactor)
```

This step ensures ODIS cannot see the actual identifier.

**4. Query ODIS**

Send the blinded hash to ODIS operators:

```typescript
const { obfuscatedIdentifier } = await OdisUtils.Identifier.getObfuscatedIdentifier(
  plaintextIdentifier,
  identifierPrefix,
  issuerAddress,
  authSigner,
  serviceContext
);
```

**5. ODIS Processing**

Each ODIS operator:

1. Receives the blinded hash
2. Computes a partial signature using their key share
3. Returns the blinded partial signature

**6. Combine Signatures**

When k signatures are received:

1. Signatures are combined into a full signature
2. This is the "blinded pepper signature"

**7. Unblind the Signature**

The client unblinds the signature:

```
unblinedSignature = unblind(blindedPepperSignature, blindingFactor)
```

**8. Generate the Pepper**

Extract the pepper from the unblinded signature:

```
pepper = first13Chars(sha256(unblindedSignature))
```

**9. Create Obfuscated Identifier**

Combine the hashed identifier with the pepper:

```
obfuscatedIdentifier = sha3(hashedIdentifier + "__" + pepper)
```

**Final Formula:**

```
obfuscatedIdentifier = sha3(sha3("{prefix}://{plaintext}") + "__" + pepper)
```

#### Verification

Results from ODIS can be verified against the service's public key, which is shared with users through the client library.

### Registration & Lookup Flow

#### Registration Flow

The process of creating an attestation mapping:

```
┌──────┐                ┌────────┐              ┌──────┐              ┌──────────────┐
│ User │                │ Issuer │              │ ODIS │              │  Blockchain  │
└───┬──┘                └───┬────┘              └───┬──┘              └──────┬───────┘
    │                       │                       │                        │
    │ 1. Request Verification│                      │                        │
    ├──────────────────────>│                       │                        │
    │                       │                       │                        │
    │ 2. Verify Ownership   │                       │                        │
    │    (SMS, OAuth, etc.) │                       │                        │
    │<─────────────────────>│                       │                        │
    │                       │                       │                        │
    │                       │ 3. Query for Pepper   │                        │
    │                       ├──────────────────────>│                        │
    │                       │                       │                        │
    │                       │ 4. Return Pepper      │                        │
    │                       │<──────────────────────┤                        │
    │                       │                       │                        │
    │                       │ 5. Register Attestation                        │
    │                       ├───────────────────────────────────────────────>│
    │                       │                       │                        │
    │                       │ 6. Attestation Registered                      │
    │                       │<───────────────────────────────────────────────┤
    │                       │                       │                        │
    │ 7. Confirmation       │                       │                        │
    │<──────────────────────┤                       │                        │
```

**Detailed Steps:**

1. **User Requests Verification**
   * User provides their identifier and wallet address to an issuer
   * User indicates they want to create an attestation
2. **Issuer Verifies Ownership**
   * Phone: Send SMS with verification code
   * Twitter: OAuth flow
   * Email: Verification link
   * Custom: Any verification method the issuer chooses
3. **Issuer Queries ODIS**
   * Issuer sends blinded identifier to ODIS
   * Consumes issuer's ODIS quota
   * Receives pepper for obfuscation
4. **Issuer Computes Obfuscated Identifier**
   * Combines hashed identifier with pepper
   * Creates the final obfuscated identifier
5.  **Register Attestation On-Chain**

    ```typescript
    await federatedAttestationsContract
      .registerAttestationAsIssuer(
        obfuscatedIdentifier,
        userAccountAddress,
        attestationVerifiedTime
      )
      .send();
    ```
6. **Gas Payment Options**
   * **Issuer Pays:** Issuer executes transaction with their gas
   * **User Pays:** Issuer signs attestation, user submits transaction

#### Lookup Flow

The process of finding addresses from identifiers:

```
┌─────────────┐          ┌──────┐          ┌──────────────┐
│ Application │          │ ODIS │          │  Blockchain  │
└──────┬──────┘          └───┬──┘          └──────┬───────┘
       │                     │                    │
       │ 1. Query for Pepper │                    │
       ├────────────────────>│                    │
       │                     │                    │
       │ 2. Return Pepper    │                    │
       │<────────────────────┤                    │
       │                     │                    │
       │ 3. Lookup Attestations                   │
       ├─────────────────────────────────────────>│
       │                     │                    │
       │ 4. Return Addresses │                    │
       │<─────────────────────────────────────────┤
```

**Detailed Steps:**

1.  **Get Obfuscated Identifier**

    ```typescript
    const { obfuscatedIdentifier } = await OdisUtils.Identifier.getObfuscatedIdentifier(
      plaintextIdentifier,
      identifierPrefix,
      lookupAddress,
      authSigner,
      serviceContext
    );
    ```
2.  **Query FederatedAttestations Contract**

    ```typescript
    const attestations = await federatedAttestationsContract.lookupAttestations(
      obfuscatedIdentifier,
      [issuer1Address, issuer2Address, ...]
    );
    ```
3.  **Process Results**

    ```typescript
    // Returns arrays for each issuer
    const {
      countsPerIssuer,  // Number of attestations per issuer
      accounts,          // Account addresses
      signers,           // Signer addresses
      issuedOns,         // Verification timestamps
      publishedOns       // Registration timestamps
    } = attestations;
    ```

#### Multi-Issuer Lookup

Applications can query multiple issuers simultaneously:

```typescript
const trustedIssuers = [
  "0x6549aF2688e07907C1b821cA44d6d65872737f05", // Kaala
  "0x388612590F8cC6577F19c9b61811475Aa432CB44"  // Libera
];

const attestations = await federatedAttestationsContract.lookupAttestations(
  obfuscatedIdentifier,
  trustedIssuers
);

// Results are ordered by issuer
// attestations.countsPerIssuer[0] = number of attestations from Kaala
// attestations.countsPerIssuer[1] = number of attestations from Libera
```

#### Trust Model

Applications must decide which issuers to trust:

**Single Issuer Trust:**

* Trust only attestations from your own issuer
* Maximum control over verification quality
* Limited to your own user base

**Multiple Issuer Trust:**

* Trust attestations from multiple issuers
* Broader coverage and interoperability
* Must evaluate each issuer's verification quality

**Consensus-Based Trust:**

* Require attestations from multiple issuers
* Higher confidence in verification
* Reduced coverage (fewer users will have multiple attestations)

### Security Considerations

#### Privacy Assumptions

Self Connect's privacy model assumes:

1. **ODIS Operators:** At least k operators remain honest
2. **Rate Limiting:** Quota system prevents mass scraping
3. **Blinding:** Client-side blinding is implemented correctly
4. **No Collusion:** Attackers don't control k or more ODIS operators

**If Assumptions Hold:**

* Identifiers cannot be reverse-engineered from blockchain data
* Rainbow table attacks are prohibitively expensive
* User privacy is preserved

**If Assumptions Fail:**

* If k operators are compromised: Peppers for any identifier can be computed
* If rate limiting is bypassed: Rainbow tables become feasible

#### Sybil Resistance

Self Connect provides Sybil resistance through:

1. **Verification Requirements:** Users must prove ownership of identifiers
2. **Issuer Quality:** Applications choose issuers with strong verification
3. **Costly Registration:** ODIS quota costs make mass fake registrations expensive
4. **Unique Identifiers:** Phone numbers and verified social accounts are limited per person

**Limitations:**

* Relies on issuer verification quality
* Some identifiers (email) are easier to create in bulk
* Applications must choose issuers carefully

#### Comparison: ASv1 vs. Self Connect

| Aspect                    | ASv1                              | Self Connect                 |
| ------------------------- | --------------------------------- | ---------------------------- |
| **Verification**          | 3 randomly selected validators    | Issuer (flexible method)     |
| **Trust Model**           | Single root: Validator collective | Multiple roots: Each issuer  |
| **Verification Quality**  | Standardized across network       | Varies by issuer             |
| **Flexibility**           | Phone numbers only                | Any identifier type          |
| **Censorship Resistance** | Validator majority                | Multi-issuer selection       |
| **Scalability**           | Limited by validator overhead     | Scales with issuer ecosystem |

#### Best Practices

1. **Choose Trusted Issuers**
   * Evaluate verification methods
   * Check issuer reputation
   * Monitor issuer behavior
2. **Implement Rate Limiting**
   * Limit lookup frequency per user
   * Monitor for abnormal query patterns
3. **Validate Results**
   * Check timestamps for recency
   * Verify issuer addresses
   * Handle multiple attestations appropriately
4. **Secure Key Management**
   * Protect issuer private keys
   * Use hardware security modules for production
   * Implement key rotation procedures
5. **Monitor ODIS Quota**
   * Set up alerts for low quota
   * Implement automatic top-ups
   * Track quota usage patterns
