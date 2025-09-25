# Happy Birthday Example

This example demonstrates the V2 [Happy Birthday contract](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/HappyBirthday.sol) that distributes USDC to users on their birthday, with document type bonuses and support for both E-Passport and EU ID Card verification.

## Birthday-Specific Features

* **Birthday Window Validation:** Configurable time window around user's birthday
* **Document Type Bonuses:** Different reward multipliers for E-Passport vs EU ID cards
* **Date Processing:** Simplified handling of pre-extracted V2 date attributes
* **One-time Claims:** Nullifier prevents multiple birthday claims

For standard V2 integration patterns, see [Basic Integration Guide](broken-reference).

### State Variables

```solidity
/// @notice USDC token contract
IERC20 public immutable usdc;

/// @notice Default: 50 USDC (6 decimals)
uint256 public claimableAmount = 50e6;

/// @notice Bonus multiplier for EU ID card users (in basis points)
uint256 public euidBonusMultiplier = 200; // 200% = 2x bonus

/// @notice Bonus multiplier for E-Passport users (in basis points) 
uint256 public passportBonusMultiplier = 150; // 150% = 1.5x bonus

/// @notice Birthday claim window (default: 1 day)
uint256 public claimableWindow = 1 days;

/// @notice Tracks users who have claimed to prevent double claims
mapping(uint256 nullifier => bool hasClaimed) public hasClaimed;

/// @notice Verification config ID for identity verification
bytes32 public verificationConfigId;

uint256 public constant BASIS_POINTS = 10000;
```

### Birthday Verification Logic

The core birthday validation uses V2's pre-extracted date format:

**Birthday Verification Hook:**

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory /* userData */
) internal override {
    // Birthday-specific validations
    if (hasClaimed[output.nullifier]) revert AlreadyClaimed();
    if (!_isWithinBirthdayWindow(output.attestationId, output.dateOfBirth)) {
        revert NotWithinBirthdayWindow();
    }

    // Calculate bonus based on document type
    uint256 finalAmount = claimableAmount;
    if (output.attestationId == AttestationId.EU_ID_CARD) {
        finalAmount = (claimableAmount * euidBonusMultiplier) / BASIS_POINTS;
    } else if (output.attestationId == AttestationId.E_PASSPORT) {
        finalAmount = (claimableAmount * passportBonusMultiplier) / BASIS_POINTS;
    }

    // Process birthday claim
    hasClaimed[output.nullifier] = true;
    address recipient = address(uint160(output.userIdentifier));
    usdc.safeTransfer(recipient, finalAmount);
    
    emit USDCClaimed(recipient, finalAmount, output.attestationId);
}
```

### V2 Simplified Birthday Verification

The V2 implementation dramatically simplifies birthday verification by using pre-extracted date attributes:

```solidity
function _isWithinBirthdayWindow(
    bytes32 attestationId, 
    string memory dobFromProof
) internal view returns (bool) {
    // DOB comes in format "DD-MM-YY" from the V2 proof system
    bytes memory dobBytes = bytes(dobFromProof);
    require(dobBytes.length == 8, "Invalid DOB format"); // "DD-MM-YY" = 8 chars

    // Extract day and month from "DD-MM-YY" format
    string memory day = Formatter.substring(dobFromProof, 0, 2);   // DD
    string memory month = Formatter.substring(dobFromProof, 3, 5); // MM (skip hyphen)

    // Create birthday in current year (format: YYMMDD)
    string memory dobInThisYear = string(abi.encodePacked("25", month, day));
    uint256 dobInThisYearTimestamp = Formatter.dateToUnixTimestamp(dobInThisYear);

    uint256 currentTime = block.timestamp;
    uint256 timeDifference;

    if (currentTime > dobInThisYearTimestamp) {
        timeDifference = currentTime - dobInThisYearTimestamp;
    } else {
        timeDifference = dobInThisYearTimestamp - currentTime;
    }

    return timeDifference <= claimableWindow;
}
```

### Document Type Bonuses

The V2 implementation provides different bonuses based on document type:

```solidity
// EU ID Card users get 2x bonus
if (output.attestationId == AttestationId.EU_ID_CARD) {
    finalAmount = (claimableAmount * 200) / 10000; // 200% = 2x
}
// E-Passport users get 1.5x bonus  
else if (output.attestationId == AttestationId.E_PASSPORT) {
    finalAmount = (claimableAmount * 150) / 10000; // 150% = 1.5x
}
```

### Administrative Functions

```solidity
// Update claimable amount
function setClaimableAmount(uint256 newAmount) external onlyOwner {
    uint256 oldAmount = claimableAmount;
    claimableAmount = newAmount;
    emit ClaimableAmountUpdated(oldAmount, newAmount);
}

// Update birthday claim window
function setClaimableWindow(uint256 newWindow) external onlyOwner {
    uint256 oldWindow = claimableWindow;
    claimableWindow = newWindow;
    emit ClaimableWindowUpdated(oldWindow, newWindow);
}

// Update EU ID bonus multiplier
function setEuidBonusMultiplier(uint256 newMultiplier) external onlyOwner {
    uint256 oldMultiplier = euidBonusMultiplier;
    euidBonusMultiplier = newMultiplier;
    emit EuidBonusMultiplierUpdated(oldMultiplier, newMultiplier);
}

// Set verification config ID
function setConfigId(bytes32 configId) external onlyOwner {
    verificationConfigId = configId;
}

// Withdraw USDC from contract
function withdrawUSDC(address to, uint256 amount) external onlyOwner {
    usdc.safeTransfer(to, amount);
}
```

### Birthday Contract Benefits

**V2 Date Simplification:** Direct access to `output.dateOfBirth` eliminates complex parsing **Multi-Document Rewards:** Different bonus structures for passport vs EU ID card users **Flexible Windows:** Configurable birthday claim periods **Admin Controls:** Owner can adjust amounts, windows, and bonuses

For verification configuration setup, see [Hub Verification Process](../technical-docs/verification-in-the-identityverificationhub.md#v2-enhanced-verifications).

### Configuration Management

The contract includes methods for managing verification configuration:

```solidity
// Override to provide configId for verification
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier,
    bytes memory userDefinedData
) public view override returns (bytes32) {
    return verificationConfigId;
}
```

### Example Usage

1. **Deploy Contract:** With hub address, scope, and USDC token address
2. **Set Configuration:** Call `setConfigId()` with your verification config ID or use `setupVerificationConfig()` pattern
3. **Fund Contract:** Transfer USDC to contract for distribution
4. **User Claims:** Users verify identity and automatically receive birthday bonus
5. **Document Bonuses:** EU ID card users get 2x, passport users get 1.5x the base amount

## Related Documentation

* [Basic Integration Guide](broken-reference) - Core V2 integration patterns
* [Identity Attributes](broken-reference) - Date handling and attribute access
* [Airdrop Example](airdrop-example.md) - Registration-based verification example
* [Hub Verification Process](../technical-docs/verification-in-the-identityverificationhub.md) - Verification configuration
