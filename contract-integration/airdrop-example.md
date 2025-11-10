# Airdrop Example

This example demonstrates V2 contract integration using the [Airdrop contract](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/Airdrop.sol), which supports both E-Passport and EU ID Card verification with registration/claim phases and Merkle tree token distribution.

### Airdrop-Specific Features

This contract demonstrates:

* **Two-phase distribution:** Registration → Claim separation
* **Merkle tree allocation:** Fair token distribution
* **Multi-document registration:** Both E-Passport and EU ID cards supported
* **Anti-duplicate measures:** Nullifier and user identifier tracking

### Registration Logic

The registration phase validates user eligibility and prevents duplicate registrations:

**Key Validations:**

* Registration phase must be open
* Nullifier hasn't been used (prevents same document registering twice)
* Valid user identifier provided
* User identifier hasn't already registered (prevents address reuse)

### State Variables

```solidity
/// @notice Maps nullifiers to user identifiers for registration tracking
mapping(uint256 nullifier => uint256 userIdentifier) internal _nullifierToUserIdentifier;

/// @notice Maps user identifiers to registration status
mapping(uint256 userIdentifier => bool registered) internal _registeredUserIdentifiers;

/// @notice Tracks addresses that have claimed tokens
mapping(address => bool) public claimed;

/// @notice ERC20 token to be airdropped
IERC20 public immutable token;

/// @notice Merkle root for claim validation
bytes32 public merkleRoot;

/// @notice Phase control
bool public isRegistrationOpen;
bool public isClaimOpen;

/// @notice Verification config ID for identity verification
bytes32 public verificationConfigId;
```

For standard V2 integration patterns (constructor, getConfigId), see [Basic Integration Guide](broken-reference/).

**Registration Verification Hook:**

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory /* userData */
) internal override {
    // Airdrop-specific validations
    if (!isRegistrationOpen) revert RegistrationNotOpen();
    if (_nullifierToUserIdentifier[output.nullifier] != 0) revert RegisteredNullifier();
    if (output.userIdentifier == 0) revert InvalidUserIdentifier();
    if (_registeredUserIdentifiers[output.userIdentifier]) revert UserIdentifierAlreadyRegistered();

    // Register user for airdrop
    _nullifierToUserIdentifier[output.nullifier] = output.userIdentifier;
    _registeredUserIdentifiers[output.userIdentifier] = true;

    emit UserIdentifierRegistered(output.userIdentifier, output.nullifier);
}
```

### Claim Function Implementation

```solidity
function claim(uint256 index, uint256 amount, bytes32[] memory merkleProof) external {
    if (isRegistrationOpen) {
        revert RegistrationNotClosed();
    }
    if (!isClaimOpen) {
        revert ClaimNotOpen();
    }
    if (claimed[msg.sender]) {
        revert AlreadyClaimed();
    }
    if (!_registeredUserIdentifiers[uint256(uint160(msg.sender))]) {
        revert NotRegistered(msg.sender);
    }

    // Verify the Merkle proof
    bytes32 node = keccak256(abi.encodePacked(index, msg.sender, amount));
    if (!MerkleProof.verify(merkleProof, merkleRoot, node)) revert InvalidProof();

    // Mark as claimed and transfer tokens
    claimed[msg.sender] = true;
    token.safeTransfer(msg.sender, amount);

    emit Claimed(index, msg.sender, amount);
}
```

### Configuration Management

The contract includes methods for managing verification configuration:

```solidity
// Set verification config ID
function setConfigId(bytes32 configId) external onlyOwner {
    verificationConfigId = configId;
}

// Override to provide configId for verification
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier,
    bytes memory userDefinedData
) public view override returns (bytes32) {
    return verificationConfigId;
}
```

### Administrative Functions

```solidity
// Set Merkle root for claim validation
function setMerkleRoot(bytes32 newMerkleRoot) external onlyOwner;

// Update verification scope
function setScope(uint256 newScope) external onlyOwner;

// Phase control
function openRegistration() external onlyOwner;
function closeRegistration() external onlyOwner;
function openClaim() external onlyOwner;
function closeClaim() external onlyOwner;
```

### Airdrop Flow

1. **Deploy:** Owner deploys with hub address, scope, and token
2. **Configure:** Set verification config ID and Merkle root using `setConfigId()` and `setMerkleRoot()`
3. **Open Registration:** Users prove identity to register
4. **Close Registration:** Move to claim phase
5. **Open Claims:** Registered users claim via Merkle proofs
6. **Distribution Complete:** Tokens distributed to verified users

For verification configuration setup, see [Hub Verification Process](../technical-docs/verification-in-the-identityverificationhub.md#v2-enhanced-verifications).
