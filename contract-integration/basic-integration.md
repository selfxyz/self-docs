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

### Integration Implementation

Extend `SelfVerificationRoot` and implement the required methods `customVerificationHook` and `getConfigId`:

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
        // Set up your verification configuration (see setVerificationConfigV2 below)
    }

    // Required: Override to provide configId for verification
    function getConfigId(
        bytes32 destinationChainId,
        bytes32 userIdentifier, 
        bytes memory userDefinedData
    ) public view override returns (bytes32) {
        // Return your app's configuration ID
        // This can be static or dynamic based on your needs
        return configId;
    }

    // Override to handle successful verification
    function customVerificationHook(
        ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
        bytes memory userData
    ) internal virtual override {
        // Your custom logic here
        // output contains verified user data including:
        // - attestationId (E_PASSPORT or EU_ID_CARD)
        // - userIdentifier, nullifier, name, nationality, etc.
        
        // Example: Check document type
        if (output.attestationId == AttestationId.E_PASSPORT) {
            // Handle passport verification
            require(bytes(output.nationality).length > 0, "Nationality required");
        } else if (output.attestationId == AttestationId.EU_ID_CARD) {
            // Handle EU ID card verification
            require(bytes(output.issuingState).length > 0, "Issuing state required");
        }
    }
}
```

For more details on the server-side architecture of Self, please refer to [the detailed architecture documentation](../technical-docs/architecture.md).

### Verification Flow

1. **Application Integration:**\
   In your third-party application (which integrates our SDK), specify the target contract address for verification.
2. **Document Scan:**\
   User scans passport/EU ID card via mobile app. The document data, along with the contract address specified by your application, is sent to our TEE.
3. **TEE Processing:**\
   Trusted Execution Environment generates zero-knowledge proof and calls your contract.
4. **Contract Execution:**\
   Your contract's `verifySelfProof` method is called with proof data, which triggers verification in the hub and then calls your `customVerificationHook` callback with structured output data.

### Verification Configuration

For interactive configuration management, use the [Self Configuration Tools](https://tools-mu-one.vercel.app/) to easily create and validate your `VerificationConfigV2` settings.

```solidity
import {IIdentityVerificationHubV2} from "@selfxyz/contracts/contracts/interfaces/IIdentityVerificationHubV2.sol";
import {SelfStructs} from "@selfxyz/contracts/contracts/libraries/SelfStructs.sol";

function setupVerificationConfig() external onlyOwner {
    IIdentityVerificationHubV2 hub = IIdentityVerificationHubV2(_identityVerificationHubV2);
    
    SelfStructs.VerificationConfigV2 memory config = SelfStructs.VerificationConfigV2({
        olderThanEnabled: true,
        olderThan: 18, // Require users to be at least 18 years old
        forbiddenCountriesEnabled: false,
        forbiddenCountriesListPacked: [uint256(0), uint256(0), uint256(0), uint256(0)],
        ofacEnabled: [false, false, false] // Disable OFAC checks
    });
    
    configId = hub.setVerificationConfigV2(config);
}
```

### Attribute Access

V2 provides direct access to verified identity attributes. See [Utilize Identity Attributes](utilize-passport-attributes.md) for comprehensive attribute handling patterns.

```solidity
function customVerificationHook(
    ISelfVerificationRoot.GenericDiscloseOutputV2 memory output,
    bytes memory userData
) internal override {
    // Document type validation
    if (output.attestationId == AttestationId.E_PASSPORT) {
        require(bytes(output.nationality).length > 0, "Nationality required");
    } else if (output.attestationId == AttestationId.EU_ID_CARD) {
        require(bytes(output.issuingState).length > 0, "Issuing state required");
    }
    
    // Your application logic here...
}
```

For detailed verification configuration options (age, country restrictions, OFAC), see [Hub Verification Process](../verification-in-the-identityverificationhub.md).

### Deployment Configuration

**Hub Addresses** (see [Deployed Contracts](deployed-contracts.md)):
- Celo Mainnet: `0x77117D60eaB7C044e785D68edB6C7E0e134970Ea`
- Celo Testnet: `0x68c931C9a534D37aa78094877F46fE46a49F1A51`

**Scope Generation:**
```typescript
import { hashEndpointWithScope } from "@selfxyz/core";
const scope = hashEndpointWithScope(contractAddress, 'your-scope-value');
```

You can use the [Self Configuration Tools](https://tools-mu-one.vercel.app/) to calculate the scope for your contract.

### Reference Implementations

**[Airdrop Contract](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/Airdrop.sol)**: Token distribution with registration/claim phases, nullifier management, and Merkle tree validation supporting both E-Passport and EU ID cards

**[Happy Birthday Contract](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/HappyBirthday.sol)**: USDC distribution on birthdays with document type bonuses (EU ID cards get higher bonus)

**[Identity NFT](https://github.com/selfxyz/self/blob/main/contracts/contracts/example/SelfIdentityERC721.sol)**: NFT minting with complete on-chain identity attribute storage using the V2 structured output

### Documentation Map

- **[Hub Verification Process](../verification-in-the-identityverificationhub.md)** - Detailed V2 verification flow and configuration
- **[Identity Attributes](utilize-passport-attributes.md)** - Comprehensive attribute access patterns
- **[SDK Configuration](frontend-configuration.md)** - Frontend integration setup
- **[Deployed Contracts](deployed-contracts.md)** - Network addresses and integration notes
- **Examples:** [Airdrop](airdrop-example.md) | [Birthday](happy-birthday-example.md)