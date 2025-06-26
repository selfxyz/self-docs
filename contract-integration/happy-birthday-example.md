# Happy Birthday Example

In the Airdrop example, we showed how to customize verification parameters when integrating Self. In this example, we will demonstrate how to use the attributes contained in a passport to build [the "happy-birthday" app](https://github.com/selfxyz/happy-birthday/blob/main/contracts/contracts/HappyBirthday.sol).

Breakdown of the repo:

* [verify a Self identity proof in the contracts](https://github.com/selfxyz/happy-birthday/blob/main/contracts/contracts/HappyBirthday.sol)
* [calling the contracts](https://github.com/selfxyz/happy-birthday/blob/main/frontend/pages/api/verify.ts)
* [display the QR code](https://github.com/selfxyz/happy-birthday/blob/main/frontend/app/page.tsx)

The **happy-birthday** application checks the user's birthday recorded in their passport and, if the birthday is within ±5 days of the current block timestamp, awards the user with USDC.

### Importing Required Files

Just like in the Airdrop example, start by importing the following files:

```solidity
import {SelfVerificationRoot} from "@selfxyz/contracts/contracts/abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts/interfaces/ISelfVerificationRoot.sol";
```

Since the passport attributes are packed into the proof as `revealedData_packed`, import these libraries to extract and handle the data:

```solidity
import {SelfCircuitLibrary} from "@selfxyz/contracts/contracts/libraries/SelfCircuitLibrary.sol";
```

### Verifications in happy-birthday

The happy-birthday app performs the following verifications:

* **Scope Check:**\
  Confirm that the proof was generated specifically for this app.
* **Attestation ID Check:**\
  Ensure the proof was created using the correct attestation ID for the document used by the app.
* **Nullifier Registration and Check:**\
  Prevent multiple claims by registering and verifying the nullifier. The nullifier is a unique value that corresponds one-to-one with the document without revealing any details.
* **Proof Verification via IdentityVerificationHub:**\
  Validate the proof itself, including additional verifications (e.g., olderThan, forbiddenCountries, and OFAC) as set by the contract.
* **Birthday Verification:**\
  Verify that the user's date of birth (extracted from the proof) is within ±5 days of the current `block.timestamp`.

### Overriding the `verifySelfProof` Function

Below is an example implementation of `verifySelfProof` for the happy-birthday app:

```solidity
function verifySelfProof(
    IVcAndDiscloseCircuitVerifier.VcAndDiscloseProof memory proof
)
    public
    override
{
    if (_nullifiers[proof.pubSignals[NULLIFIER_INDEX]]) {
        revert RegisteredNullifier();
    }

    super.verifySelfProof(proof);

    if (_isWithinBirthdayWindow(
           getRevealedDataPacked(proof.pubSignals)
        )
    ) {
        _nullifiers[proof.pubSignals[NULLIFIER_INDEX]] = true;
        usdc.safeTransfer(address(uint160(proof.pubSignals[USER_IDENTIFIER_INDEX])), CLAIMABLE_AMOUNT);
        emit USDCClaimed(address(uint160(proof.pubSignals[USER_IDENTIFIER_INDEX])), CLAIMABLE_AMOUNT);
    } else {
        revert("Not eligible: Not within 5 days of birthday");
    }
}

```



### Extracting and Comparing the Date of Birth

The birthday information is packed into `revealedData_packed` in the proof. To verify the birthday, follow these steps:

1. **Convert the Packed Data:**\
   Convert `revealedData_packed` into a single `bytes` array using the `Formatter.fieldElementsToBytes` function.
2. **Extract the Date of Birth:**\
   Use the `CircuitAttributeHandler.getDateOfBirth` function to extract the date of birth from the byte array.
3. **Reformat the Date:**\
   The extracted date is in the format `DD-MM-YY`, but the `Formatter.dateToUnixTimestamp` function expects a `YYMMDD` string. Rearrange the data accordingly.\
   &#xNAN;_(In this example, the year is hard-coded as "25" to represent the birthday in 2025 at midnight.)_
4. **Compare with the Current Time:**\
   Convert the reformatted date to a timestamp and compare it with the current `block.timestamp` to check if the difference is within ±5 days.



Below is the helper function `_isWithinBirthdayWindow` that implements this logic:

```solidity
function _isWithinBirthdayWindow(
    // Accepts the revealedData_packed from the proof.
    uint256[3] memory revealedDataPacked
) 
    internal 
    view 
    returns (bool) 
{
    // Convert the field elements into a single bytes array.
    string memory dob = SelfCircuitLibrary.getDateOfBirth(revealedDataPacked);

    bytes memory dobBytes = bytes(dob);
    bytes memory dayBytes = new bytes(2);
    bytes memory monthBytes = new bytes(2);

    dayBytes[0] = dobBytes[0];
    dayBytes[1] = dobBytes[1];

    monthBytes[0] = dobBytes[3];
    monthBytes[1] = dobBytes[4];
    
    string memory day = string(dayBytes);
    string memory month = string(monthBytes);
    string memory dobInThisYear = string(abi.encodePacked("25", month, day));

    uint256 dobInThisYearTimestamp = SelfCircuitLibrary.dateToTimestamp(dobInThisYear);

    // Get the current block.timestamp and compute the absolute time difference.
    uint256 currentTime = block.timestamp;
    uint256 timeDifference = currentTime > dobInThisYearTimestamp
        ? currentTime - dobInThisYearTimestamp
        : dobInThisYearTimestamp - currentTime;
    
    // Check if the difference is within ± 5 days.
    uint256 fiveDaysInSeconds = 5 days;
    return timeDifference <= fiveDaysInSeconds;
}

```



In summary, the birthday comparison requires some data transformation after extracting the date of birth from `revealedData_packed`. If you only need to retrieve an attribute, the combination of `Formatter.fieldElementsToBytes` and the appropriate function from `CircuitAttributeHandler` will suffice.



This example demonstrates how to leverage passport attributes—specifically the date of birth—to implement custom logic in your smart contract integration. Adjust the implementation as needed for your application's requirements.
