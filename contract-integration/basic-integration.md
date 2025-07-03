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
npm install @selfxyz/contracts @openzeppelin/contracts
```

### Step 2: Create Your Contract

Extend `SelfVerificationRoot` and implement the required methods:

```solidity
import {SelfVerificationRoot} from "@selfxyz/contracts/contracts/abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts/interfaces/ISelfVerificationRoot.sol";
import {IIdentityVerificationHubV2} from "@selfxyz/contracts/contracts/interfaces/IIdentityVerificationHubV2.sol";
import {SelfStructs} from "@selfxyz/contracts/contracts/libraries/SelfStructs.sol";
import {AttestationId} from "@selfxyz/contracts/contracts/constants/AttestationId.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract ExampleV2 is SelfVerificationRoot, Ownable {
    // Your app-specific configuration ID
    bytes32 public configId;
    
    constructor(
        address _identityVerificationHubV2, // V2 Hub address
        uint256 _scope // Application-specific scope identifier
    ) 
        SelfVerificationRoot(_identityVerificationHubV2, _scope)
        Ownable(msg.sender)
    {
        // Initialize with empty configId - set it up in Step 2
    }

    // Required: Override to provide configId for verification
    function getConfigId(
        bytes32 destinationChainId,
        bytes32 userIdentifier, 
        bytes memory userDefinedData // Custom data from the qr code configuration
    ) public view override returns (bytes32) {
        // Return your app's configuration ID
        return configId;
    }

    // Override to handle successful verification
    function customVerificationHook(
        ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
        bytes memory userData
    ) internal virtual override {
        // Your custom business logic here
        // Example: Store verified user data, mint NFT, transfer tokens, etc.
        
        // Access verified data:
        // output.userIdentifier - user's unique identifier
        // output.name - verified name
        // output.nationality - verified nationality
        // output.dateOfBirth - verified birth date
        // etc.
        
        // Example: Simple verification check
        require(bytes(output.nationality).length > 0, "Nationality required");
    }
}
```

For more details on the server-side architecture of Self, please refer to [the detailed architecture documentation](../technical-docs/architecture.md).

### Step 3: Generate Configuration ID

Use the [Self Configuration Tools](https://tools.self.xyz/) to easily create your verification configuration and generate a config ID. This tool allows you to configure age requirements, country restrictions, and OFAC checks with a user-friendly interface.

Once you have your config ID from the tool, you can use it in your contract in several ways:

**Option 1: Hard-coded (simple approach)**
```solidity
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier, 
    bytes memory userDefinedData
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
    bytes32 destinationChainId,
    bytes32 userIdentifier, 
    bytes memory userDefinedData
) public view override returns (bytes32) {
    return configId;
}
```

**Option 3: Dynamic based on user context**
```solidity
mapping(bytes32 => bytes32) public userConfigIds;

function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier, 
    bytes memory userDefinedData
) public view override returns (bytes32) {
    // Use different config IDs based on user or context
    bytes32 userSpecificConfig = userConfigIds[userIdentifier];
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