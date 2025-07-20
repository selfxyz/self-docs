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

This advanced approach allows your contract to use different verification configurations based on user actions and context. Instead of having a single, fixed config ID, you can dynamically select which verification requirements to apply based on what the user is trying to do.

**Why use dynamic config IDs?**
- Different actions may require different verification levels (e.g., voting vs. general access)
- External contracts might need specific identity requirements 
- You can support multiple use cases in a single contract

**Example scenario:**
Users can join polls (actionType 0) with nationality verification, or get general verification (actionType 1) with basic identity proof.

<details>
<summary><strong>📋 View Complete Implementation</strong></summary>

**How the system works:**
The system encodes user intent in the `userData` parameter:
- **Frontend sends:** `actionType + accessCode` (e.g., actionType=0, accessCode=bytes32 value)
- **Contract receives:** Parsed action (uint8) and access code (bytes32) to determine appropriate config ID
- **Verification flows:** Different configs = different requirements

**Implementation breakdown:**
1. **Data Structure:** Mappings to connect access codes → contracts → config IDs
2. **Parsing Logic:** Extract action type and access code from frontend data
3. **Config Resolution:** Route to appropriate verification configuration
4. **Action Execution:** Perform different logic based on verification context

```solidity
// Default config ID for general verifications
bytes32 private constant DEFAULT_VERIFICATION_CONFIG_ID = 
    0x7b6436b0c98f62380866d9432c2af0ee08ce16a171bda6951aecd95ee1307d61;

// Core mappings for dynamic config system
mapping(bytes32 accessCode => address targetContract) public codeToContractAddress;
mapping(address targetContract => bytes32 configId) public configIds;
mapping(address participant => bool verified) public isVerified;

// Example interface for external contract interaction
interface ExternalContract {
    function addParticipant(address participant, bytes memory nationality) external;
}

function getConfigId(
    bytes32 _destinationChainId,
    bytes32 _userIdentifier,
    bytes memory _userDefinedData // Format: actionType + accessCode
) public view override returns (bytes32) {
    (uint8 actionCode, bytes32 accessCode) = parseUserData(_userDefinedData);
    
    if (actionCode == 0) {
        // External contract interaction - use contract-specific config
        address contractAddr = codeToContractAddress[accessCode];
        return configIds[contractAddr];
    } else if (actionCode == 1) {
        // General verification - use default config
        return DEFAULT_VERIFICATION_CONFIG_ID;
    }
    
    revert("Invalid action code");
}

function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory _output,
    bytes memory _userData // Format: actionType + accessCode
) internal override {
    (uint8 actionCode, bytes32 accessCode) = parseUserData(_userData);
    
    address participant = address(uint160(_output.userIdentifier));
    
    if (actionCode == 0) {
        // External contract interaction: call specific contract with nationality data
        address contractAddress = codeToContractAddress[accessCode];
        require(contractAddress != address(0), "Contract not found");
        
        ExternalContract externalContract = ExternalContract(contractAddress);
        externalContract.addParticipant(participant, _output.nationality);
        
    } else if (actionCode == 1) {
        // General verification: mark user as verified
        isVerified[participant] = true;
    }
}

// Enhanced parsing to handle frontend encoding variations
function parseUserData(bytes memory userData) internal pure returns (uint8 actionCode, bytes32 accessCode) {
    require(userData.length >= 33, "Invalid userData length");
    
    // Handle different encoding formats from frontend
    uint8 firstByte = uint8(userData[0]);
    if (firstByte == 0x30) {
        // ASCII '0' (48 in decimal)
        actionCode = 0;
    } else if (firstByte == 0x31) {
        // ASCII '1' (49 in decimal)
        actionCode = 1;
    } else if (firstByte == 0 || firstByte == 1) {
        // Raw bytes
        actionCode = firstByte;
    } else {
        revert("Invalid action code");
    }
    
    // Extract accessCode from remaining bytes using assembly for efficiency
    assembly {
        accessCode := mload(add(userData, 33))
    }
    
    // Handle string-encoded access codes from frontend
    accessCode = bytes32(parseUint(abi.encodePacked(accessCode)));
}

function parseUint(bytes memory _b) internal pure returns (uint256 result) {
    for (uint256 i = 1; i < _b.length; i++) {
        require(_b[i] >= 0x30 && _b[i] <= 0x39, "Invalid character");
        result = result * 10 + (uint8(_b[i]) - 48);
    }
}

// Admin functions to manage contract mappings
function setContractMapping(bytes32 _accessCode, address _contractAddress, bytes32 _configId) external {
    codeToContractAddress[_accessCode] = _contractAddress;
    configIds[_contractAddress] = _configId;
}

function removeContractMapping(bytes32 _accessCode) external {
    address contractAddress = codeToContractAddress[_accessCode];
    delete codeToContractAddress[_accessCode];
    delete configIds[contractAddress];
}
```

</details>

<details>
<summary><strong>🔧 Frontend-to-Backend Encoding Guide</strong></summary>

#### How to encode data from frontend to backend

**1. Frontend Data Preparation:**
```javascript
// Frontend prepares userData for verification
const actionType = 0; // 0 = Join Poll (dynamic config ID), 1 = Register (Default Verification)
const accessCode = '0x1234567890123456789012345678901234567890123456789012345678901234'; // 32-byte hex string

// Encode userData: actionType (1 byte) + accessCode (32 bytes)
function encodeUserData(actionType, accessCode) {
    // Convert actionType to 2-digit hex string (e.g., 0 -> '00', 1 -> '01')
    const actionHex = actionType.toString(16).padStart(2, '0');
    
    // Remove '0x' prefix from accessCode if present
    const cleanAccessCode = accessCode.replace(/^0x/, '');
    
    // Concatenate: actionType (1 byte) + accessCode (32 bytes)
    return `0x${actionHex}${cleanAccessCode}`;
}

const userDefinedData = encodeUserData(actionType, accessCode);
// Result: "0x001234567890123456789012345678901234567890123456789012345678901234"
```

**2. Backend Parsing Process:**
```solidity
// The parseUserData function handles different encoding formats:

function parseUserData(bytes memory _userData) internal pure returns (uint8 actionCode, bytes32 accessCode) {
    require(_userData.length >= 33, "Invalid userData length");
    
    // Parse first byte for action type
    uint8 firstByte = uint8(_userData[0]);
    if (firstByte == 0x30) {        // ASCII '0' (48)
        actionCode = 0;
    } else if (firstByte == 0x31) { // ASCII '1' (49)  
        actionCode = 1;
    } else if (firstByte == 0 || firstByte == 1) { // Raw bytes
        actionCode = firstByte;
    } else {
        revert("Invalid action code");
    }
    
    // Extract accessCode from remaining bytes (position 33+)
    assembly {
        accessCode := mload(add(_userData, 33))
    }
    
    // Convert string-encoded access code to number
    accessCode = bytes32(parseUint(abi.encodePacked(accessCode)));
}
```

**3. Config ID Resolution:**
- **Action Code 0**: `accessCode → contractAddress → configId`
- **Action Code 1**: Returns `DEFAULT_VERIFICATION_CONFIG_ID`

</details>

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