# Frontend Configuration

Below is an example of how to configure the QR code for on-chain verification using our SDK:

```typescript
const selfApp = new SelfAppBuilder({
        appName: "Self Example App",
        scope: "Self-Example-App",
        endpoint: "YOUR_DEPLOYED_CONTRACT_ADDRESS",
        endpointType: "staging_celo",
        logoBase64: logo,
        userId: address,
        userIdType: "hex",
        disclosures: { 
            date_of_birth: true,
        },
        devMode: true,
} as Partial<SelfApp>).build();
```



If you are using our QRCode SDK to perform on-chain verification directly, you will need to configure the following parameters:

**- `endpoint`**

This should be the address of the deployed smart contract where verification will take place.

**- `endpointType`**

Set this based on the target network:

* Use `"celo"` for Celo Mainnet
* Use `"staging_celo"` for Celo Alfajores Testnet

**- `disclosures`**

Configure this to disclose all the attributes that the smart contract is expected to verify.\
Only the disclosed attributes can be verified on-chain, so this configuration must exactly match what the contract expects.
