# Working with userDefinedData

The `userDefinedData` is mainly used in the frontend to allow users to pass in additional information during the time of verification.&#x20;

{% hint style="warning" %}
The `userDefinedData` passed in the QRCode gets converted from a **string** to **bytes**. You will have to convert it back from bytes to a string again and work on top of that.&#x20;
{% endhint %}

## Setting a config

When setting a config just creating the config is not enough. You should register the config with the hub and this method will also return the config id.&#x20;

```solidity
//internal map that stores from hash(data) -> configId
mapping(uint256 => uint256) public configs;

function setConfig(
    string memory configDesc, 
    SelfUtils.UnformattedVerificationConfigV2 config
) public {
    //create the key
    uint256 key = uint256(keccak256(bytes(configDesc)));
    //create the hub compliant config struct
    SelfStructs.VerificationConfigV2 verificationConfig = 
        SelfUtils.formatVerificationConfigV2(_verificationConfig);
    //register and get the id
    uint256 verificationConfigId = 
        IIdentityVerificationHubV2(identityVerificationHubV2Address)
        .setVerificationConfigV2(verificationConfig);
    //set it in the key
    configs[key] = verificationConfigId;
}
```

### Change the `getConfigId` in the `SelfVerificationRoot`

Now we just have to change the `getConfigId` that returns the config ids from this map. This is pretty simple as now we just have to hash the existing bytes:&#x20;

```solidity
function getConfigId(
    bytes32 destinationChainId,
    bytes32 userIdentifier,
    bytes memory userDefinedData
) public view virtual returns (bytes32) {
    //the string is already converted to bytes
    uint256 key = keccak256(userDefinedData);
    return configs[key];
}
```
