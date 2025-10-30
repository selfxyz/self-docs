---
icon: hand-holding-medical
---

# Troubleshooting

## Common errors while verifying proof

### ScopeMismatch

There is a mismatch of the scope value between the smart contract and the front end. Check the `scopeSeed` value that was used to deploy the smart contract (in your .env/deployment script). Then, check the scope value you are passing into the `SelfAppBuilder` object you are building on your frontend (e.g. in your page.tsx). Ensure the `scope` string in `SelfAppBuilder` is the same as the `scopeSeed` used in your contract.

You should also ensure that the `endpoint` used for `SelfAppBuilder` is in lowercase (unchecksum/not in checksum format), if you are using an onchain contract address.

### Invalid 'to' address

There is a mismatch between `endpoint` and `endpoint-type` in the `SelfAppBuilder` object.&#x20;

* If `endpointType` is `celo` , then `endpoint` value must be the contract address for a contract deployed on Celo Mainnet.
* If `endpointType` is `celo-staging` , then `endpoint` value must be the contract address for a contract deployed on Celo Sepolia Testnet.
* If `endpointType` is `https` , then `endpoint` value must be a http endpoint.
* If `endpointType` is `https-staging` , then `endpoint` value must be a http endpoint. `https-staging` is to be used for verifying mock documents.

### -32000, "message" : "execution reverted"

This is a generic error message thrown when something is wrong with your smart contract logic. This can be caused by a wide variety of factors. Try:

* Checking that you deployed your contract with the [correct Hub address.](../contract-integration/deployed-contracts.md)
* Ensure that your `customVerificationHook` logic is sound and contains no errors.

If the error is from a failing require check, the require statements error message will be displayed instead of the -32000 error message.

Other common error codes for interacting with smart contracts can be found in the [EIP-1474 specifications](https://eips.ethereum.org/EIPS/eip-1474).

### Invalid config ID

A common cause of this error is trying to use a Mock Passport to verify a proof on a Celo Mainnet smart contract. Mock Passports can only be used for deployments on Celo Sepolia.

### Config Mismatch

Can be caused by various different issues to do with a mismatch being found between the Verification Config specified in the smart contract and the frontend - see the [section in Basic Integration](https://docs.self.xyz/contract-integration/basic-integration#setting-verification-configs).&#x20;

### Transaction failed with error: 0xf4d678b8

If you get an error with a message like this but with differing data after the 0x, it is likely the smart contract is hitting a custom error. The error displayed is the hex selector of the custom error you have defined, and so will be different based on the name of your custom error. For example, 0xf4d678b8 is the hex selector for the custom error `InsufficientBalance`.
