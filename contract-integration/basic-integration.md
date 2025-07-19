# Basic Contract Integration Guide

This document provides an overview and integration guide for our smart contract, available as an npm package. You can install it with:

```bash
npm install @selfxyz/contracts
```

## V2 Integration

### Package Structure

The V2 package supports multiple document types with enhanced verification architecture:

```bash
.
├── abstract
│ └── SelfVerificationRoot.sol # Base impl in self verification V2
├── constants
│ ├── AttestationId.sol # Unique identifiers for identity documents (E_PASSPORT, EU_ID_CARD)
│ └── CircuitConstantsV2.sol # V2 indices for public signals in our circuits
├── interfaces # Interfaces for V2 contracts
│ ├── IDscCircuitVerifier.sol
│ ├── IIdentityRegistryV1.sol
│ ├── IIdentityRegistryIdCardV1.sol # New: EU ID Card registry interface
│ ├── IIdentityVerificationHubV2.sol # V2 hub interface
│ ├── IRegisterCircuitVerifier.sol
│ ├── ISelfVerificationRoot.sol
│ └── IVcAndDiscloseCircuitVerifier.sol
├── libraries
│ ├── SelfStructs.sol # V2 data structures for verification
│ ├── CustomVerifier.sol # Custom verification logic for different document types
│ ├── CircuitAttributeHandlerV2.sol # V2 attribute extraction
│ ├── GenericFormatter.sol # V2 output formatting
│ └── Formatter.sol # Utility functions (maintained for compatibility)
└── example
  ├── HappyBirthday.sol # Updated V2 example supporting both passports and EU ID cards
  ├── Airdrop.sol # V2 airdrop example
  └── SelfIdentityERC721.sol # NFT example with identity verification
```

## Step-by-Step Integration Guide

### Step 1: Install Dependencies

```bash
npm install @selfxyz/contracts
```

### Step 2: Create Your Contract

Extend `SelfVerificationRoot` and implement the required methods:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {SelfVerificationRoot} from "@selfxyz/contracts/contracts/abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts/interfaces/ISelfVerificationRoot.sol";

/**
 * @title ProofOfHuman
 * @notice Simple example showing how to verify human identity using Self Protocol
 */
contract ProofOfHuman is SelfVerificationRoot {
    // Store verification status for each user
    mapping(address => bool) public verifiedHumans;
    bytes32 public verificationConfigId;
    address public lastUserAddress;
    
    // Event to track successful verifications
    event VerificationCompleted(
        ISelfVerificationRoot.GenericDiscloseOutputV2 output,
        bytes userData
    );
    
    /**
     * @notice Constructor for the contract
     * @param _identityVerificationHubV2Address The address of the Identity Verification Hub V2
     * @param _scope The scope of the contract
     * @param _verificationConfigId The configuration ID for the contract
     */
    constructor(
        address _identityVerificationHubV2Address,
        uint256 _scope,
        bytes32 _verificationConfigId
    ) SelfVerificationRoot(_identityVerificationHubV2Address, _scope) {
        verificationConfigId = _verificationConfigId;
    }

    /**
     * @notice Implementation of customVerificationHook
     * @dev This function is called by onVerificationSuccess after hub address validation
     * @param output The verification output from the hub
     * @param userData The user data passed through verification
     */
    function customVerificationHook(
        ISelfVerificationRoot.GenericDiscloseOutputV2 memory _output,
        bytes memory _userData
    ) internal override {
        lastUserAddress = address(uint160(_output.userIdentifier));
        verifiedHumans[lastUserAddress] = true;

        emit VerificationCompleted(_output, _userData);
        
        // Add your custom logic here:
        // - Mint NFT to verified user
        // - Add to allowlist
        // - Transfer tokens
        // - etc.
    }

    function getConfigId(
        bytes32 _destinationChainId,
        bytes32 _userIdentifier,
        bytes memory _userDefinedData
    ) public view override returns (bytes32) {
        return verificationConfigId;
    }

    /**
     * @notice Check if an address is a verified human
     */
    function isVerifiedHuman(address _user) external view returns (bool) {
        return verifiedHumans[_user];
    }

    function setConfigId(bytes32 _configId) external {
        verificationConfigId = _configId;
    }
}
```

### Step 3: Generate Configuration ID

Use the [Self Configuration Tools](https://tools.self.xyz/) to easily create your verification configuration and generate a config ID. This tool allows you to configure age requirements, country restrictions, and OFAC checks with a user-friendly interface.

Once you have your config ID from the tool, you can use it in your contract in several ways:

**Option 1: Hard-coded**
```solidity
function getConfigId(
    bytes32 _destinationChainId,
    bytes32 _userIdentifier, 
    bytes memory _userDefinedData
) public view override returns (bytes32) {
    // Replace with your actual config ID from the tool
    return 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef;
}
```

**Option 2: Store and update via setter**
```solidity
bytes32 public configId;

function setConfigId(bytes32 _configId) external onlyOwner {
    configId = _configId;
}

function getConfigId(
    bytes32 _destinationChainId,
    bytes32 _userIdentifier, 
    bytes memory _userDefinedData
) public view override returns (bytes32) {
    return configId;
}
```

**Option 3: Dynamic based on user context**
```solidity
mapping(bytes32 => bytes32) public userConfigIds;

function getConfigId(
    bytes32 _destinationChainId,
    bytes32 _userIdentifier, 
    bytes memory _userDefinedData
) public view override returns (bytes32) {
    // Use different config IDs based on user or context
    bytes32 userSpecificConfig = userConfigIds[_userIdentifier];
    return userSpecificConfig != bytes32(0) ? userSpecificConfig : configId;
}
```

### Step 4: Deploy Your Contract

Deploy with the V2 Hub address:
- **Celo Mainnet:** `0xe57F4773bd9c9d8b6Cd70431117d353298B9f5BF`
- **Celo Testnet:** `0x68c931C9a534D37aa78094877F46fE46a49F1A51`

### Step 5: Set Up Scope

Your contract needs a proper scope for verification. You have two approaches:

**Option 1: Predict address with CREATE2 (advanced)**
```solidity
// Calculate scope before deployment using predicted address
// Use tools.self.xyz to calculate scope with your predicted contract address
```

**Option 2: Update scope after deployment (easier)**
```solidity
uint256 public scope;

function setScope(uint256 _scope) external onlyOwner {
    scope = _scope;
    // Update the scope in the parent contract
    _setScope(_scope);
}
```

After deployment, use the [Self Configuration Tools](https://tools.self.xyz/) to calculate the actual scope with your deployed contract address and update it using the setter function.

### Step 6: Configure Frontend SDK

Set up your frontend with the deployed contract address (see [Frontend Configuration](frontend-configuration.md)).

## Next Steps

- **[Identity Attributes](utilize-passport-attributes.md)** - Learn how to access verified user data
- **[Frontend Configuration](frontend-configuration.md)** - Set up your frontend integration
- **[Example Contracts](airdrop-example.md)** - See complete implementation examples