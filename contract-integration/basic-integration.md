# Basic Integration

This document provides an overview and integration guide for our smart contract, available as an npm package. You can install it with:

```
npm i @selfxyz/contracts
```

### Package Structure and Overview

The package structure and a brief explanation for each part are as follows:

```bash
.
├── abstract
│ └── SelfVerificationRoot.sol # Base impl in self verification
├── constants
│ ├── AttestationId.sol # A unique identifier assigned to the identity documents
│ └── CircuitConstants.sol # Indices for public signals in our circuits
├── interfaces # Interfaces for each contract
│ ├── IDscCircuitVerifier.sol
│ ├── IIdentityRegistryV1.sol
│ ├── IIdentityVerificationHubV1.sol
│ ├── IPassportAirdropRoot.sol
│ ├── IRegisterCircuitVerifier.sol
│ ├── ISelfVerificationRoot.sol
│ └── IVcAndDiscloseCircuitVerifier.sol
└── libraries
  ├── SelfCircuitLibrary.sol # Library for summarize all functions in the Self protocol
  ├── CircuitAttributeHandler.sol # Library to extract each attribute from public signals
  └── Formatter.sol # Utility functions to manage public signals to meaningful format
```



### Basic Integration Strategy

To leverage the apps and infrastructure provided by Self, extend the `SelfVerificationRoot` contract and use the `verifySelfProof` function. Since `SelfVerificationRoot` already contains a simple implementation, you usually won’t need to add extra code. For example:

```solidity
import {SelfVerificationRoot} from "@selfxyz/contracts/contracts/abstract/SelfVerificationRoot.sol";
import {ISelfVerificationRoot} from "@selfxyz/contracts/contracts/interfaces/ISelfVerificationRoot.sol";

contract Example is SelfVerificationRoot {
    constructor(
        address _identityVerificationHub,
        uint256 _scope, 
        uint256[] memory _attestationIds,
    ) 
        SelfVerificationRoot(
            _identityVerificationHub, // Address of our Verification Hub, e.g., "0x77117D60eaB7C044e785D68edB6C7E0e134970Ea"
            _scope, // An application-specific identifier for the integrated contract
            _attestationIds[], // The id specifying the type of document to verify (e.g., 1 for passports)
        )
    {} 
    
    function verifySelfProof(
        ISelfVerificationRoot.DiscloseCircuitProof memory proof
    ) 
        public 
        override 
    {
        super.verifySelfProof(proof);
    }
}
```

For more details on the server-side architecture of Self, please refer to [the detailed documentation on our website](../technical-docs/architecture.md).

#### How our infrastructure work

1. **Application Integration:**\
   In your third-party application (which integrates our SDK), specify the target contract address for verification.
2. **User Interaction:**\
   The user scans their passport on their device. The passport data, along with the contract address specified by your application, is sent to our TEE (Trusted Execution Environment) server.
3. **Proof Generation:**\
   Our TEE server generates a proof based on the passport information and automatically calls the specified contract address.
4. **Fixed Interface:**\
   The called contract uses a fixed interface—the `verifySelfProof` function in the abstract contract `SelfVerificationRoot`—ensuring consistency across integrations.



### Further Verifications Using Passport Attributes

In addition to verifying the validity of the passport itself, our SDK allows for customized verification using specific attributes contained within the passport. Examples include:

#### Minimum Age Check

This verification allows confirming that a user is above a certain age without revealing their exact date of birth or actual age.\
For instance, even if the user is asked to prove they are over 18, the proof will still pass on-chain even if the contract verifies whether the user is over 20.

#### Excluded Countries Check

Without disclosing the user's nationality, this check allows the contract to verify that the user does **not** belong to any of the countries in a predefined exclusion list.\
To reduce the computational load of proof generation and on-chain verification, the list of excluded countries is packed into four field values on BN254 curve.\
It is **crucial** that the country list defined in the frontend exactly matches the packed value expected by the contract to ensure consistent verification results.

#### OFAC Check

The OFAC (Office of Foreign Assets Control) list consists of individuals and entities sanctioned by the U.S. government, with whom U.S. persons and companies are generally prohibited from engaging in transactions.

Using the passport attributes, Self enables verification that a user is **not** listed on the OFAC list based on one of the following combinations of information:

* Passport number
* Name and date of birth
* Name and year of birth

**Important Note:**\
If the contract attempts to verify information that was **not requested** by the frontend (e.g., name or passport number), an error will occur.\
Make sure the attributes requested on the frontend are consistent with what the contract verifies on-chain.



### How To Set These Custom Verifications

To perform these customized verifications, you need to define a separate function that specifies which attributes should be verified.

The `SelfVerificationRoot.sol` contract already includes internal functions to handle these elements. Therefore, we recommend defining corresponding setter and getter functions as shown below:

```solidity
function setVerificationConfig(
        ISelfVerificationRoot.VerificationConfig memory newVerificationConfig
) external onlyOwner {
        _setVerificationConfig(newVerificationConfig);
}

function getVerificationConfig() external view returns (ISelfVerificationRoot.VerificationConfig memory) {
        return _getVerificationConfig();
}
```

The input values for these configuration functions will be as follows:

```solidity
struct VerificationConfig {
        bool olderThanEnabled;
        uint256 olderThan;
        bool forbiddenCountriesEnabled;
        uint256[4] forbiddenCountriesListPacked;
        bool[3] ofacEnabled;
}.
```



### Contract Deployment

If you want to perform on-chain verification using our infrastructure, please provide the following parameters to the constructor:

**- `IdentityVerificationHub`**

Specify the address of the Identity Verification Hub that we operate.\
Note that the address differs between Celo Mainnet and Celo Alfajores Testnet.\
Refer to the [**Deployed Contracts**](deployed-contracts.md) section for the correct address to use for each network.

**- `scope`**

The `scope` is a unique identifier specific to your application.\
For security purposes, the scope is calculated as follows:

$$
Poseidon("\mathrm{YourContractAddress}", "\mathrm{YourApplicationName}")
$$

So, before you actually deploy your contract, you need to calculate the future contract address.  Also, to simplify scope generation, we provide a helper function in the [@selfxyz/core](https://www.npmjs.com/package/@selfxyz/core) package:

```typescript
import { hashEndpointWithScope } from "@selfxyz/core";
import { ethers } from "hardhat";

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying contracts with the account:", deployer.address);
  
  const nonce = await ethers.provider.getTransactionCount(deployer.address);
  console.log("Account nonce:", nonce);
  
  const futureAddress = ethers.getCreateAddress({
    from: deployer.address,
    nonce: nonce
  });
  console.log("Calculated future contract address:", futureAddress);
  
  const scope = hashEndpointWithScope("futureAddress", 'Self-Example-App');
  
  // etc...
}
```

