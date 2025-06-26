# Airdrop Example

Let’s take a closer look at how to integrate our smart contract by using a concrete example. In this case, we will demonstrate how to integrate our [Airdrop contract](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/Airdrop.sol).

### Importing Required Files

In the Airdrop contract, we override the `verifySelfProof` function to add custom functionality. For this purpose, import the following files:

```solidity
import {SelfVerificationRoot} from "@selfxyz/contracts/contracts//abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts//interfaces/ISelfVerificationRoot.sol";
```

### Verification Requirements for the Airdrop

For the Airdrop use case, the following verification checks are required:

* **Scope Verification:**\
  Confirm that the proof was generated specifically for the Airdrop application by checking the scope used in the proof.
* **Attestation ID Verification:**\
  Ensure the proof was generated using the correct attestation ID corresponding to the document type intended for the Airdrop.
* **Nullifier Registration and Verification:**\
  Prevent double claims by registering and verifying a nullifier. Although the nullifier does not reveal any document details, it is uniquely tied to the document.&#x20;
* **User Identifier Registration:**\
  Verify that the proof includes a valid user identifier (in this case, the address that will receive the Airdrop).
* **Proof Verification by IdentityVerificationHub:**\
  Validate the proof itself using our IdentityVerificationHub, which also performs additional checks (e.g., olderThan, forbiddenCountries, and OFAC validations) as configured for the Airdrop.

### State Variables for Nullifier and User Identifier

Within the Airdrop contract, mappings are declared to keep track of used nullifiers and registered user identifiers:

```solidity
mapping(uint256 => uint256) internal _nullifiers;
mapping(uint256 => bool) internal _registeredUserIdentifiers;
```

### Overriding the `verifySelfProof` Function

The `verifySelfProof` function is overridden as follows to include all necessary checks:

```solidity
function verifySelfProof(
    IVcAndDiscloseCircuitVerifier.VcAndDiscloseProof memory proof
) 
    public 
    override 
{
    if (!isRegistrationOpen) {
        revert RegistrationNotOpen();
    }

    if (_nullifiers[proof.pubSignals[NULLIFIER_INDEX]] != 0) {
        revert RegisteredNullifier();
    }
    
    if (proof.pubSignals[USER_IDENTIFIER_INDEX] == 0) {
        revert InvalidUserIdentifier();
    }

    super.verifySelfProof(proof);

    _nullifiers[proof.pubSignals[NULLIFIER_INDEX]] = proof.pubSignals[USER_IDENTIFIER_INDEX];
    _registeredUserIdentifiers[proof.pubSignals[USER_IDENTIFIER_INDEX]] = true;

    emit UserIdentifierRegistered(proof.pubSignals[USER_IDENTIFIER_INDEX], proof.pubSignals[NULLIFIER_INDEX]);
}
```

