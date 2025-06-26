---
icon: passport
---

# Using mock passports

To create a mock passport, on the first screen, tap 5 times with 2 fingers on the Self card.

<figure><img src="../.gitbook/assets/IMG_AFD54FE4EA59-1 (1).jpeg" alt="" width="188"><figcaption></figcaption></figure>

This will show a screen to create a mock passport. To try it out, use [https://playground.staging.self.xyz/](https://playground.staging.self.xyz/) instead of [https://playground.self.xyz/](https://playground.self.xyz/)

When using offchain verification, pass `mockPassport` to the Self verifier as explained [here](../sdk-reference/selfbackendverifier.md).

When using onchain verification, use the [Alfajores contracts](../contract-integration/deployed-contracts.md).

To stop using a mock passport, go in Settings, then tap 5 times with 2 fingers on the empty space below the options. This will show additional options, including the Debug menu that allows going back to the first screen and scanning a new passport.

<figure><img src="../.gitbook/assets/IMG_53F184B8BCC5-1 (2).jpeg" alt="" width="188"><figcaption></figcaption></figure>

{% hint style="info" %}
Two passports registered with the same private key will give the same disclosure nullifier, thus won't be able to e.g. claim an airdrop twice.

If you want to use two passports in prod, you should backup your seed phrase then tap "Delete keychain secrets" before loading a new passport. If later you rescan the previous passport, you'll be able to pass your recovery phrase to recover the corresponding Self identity.

The next versions will support multiple IDs natively.
{% endhint %}
