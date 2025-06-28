# Architecture

## High level Architecture

Usage of the Self protocol involves two main steps: registration and disclosure. This design is somewhat similar to identity mixers such as Semaphore.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfGhkaTL8E0BZjO90mM7zUu0ZrhWswbbcfmbeqNXaJEOi225_GzefTZ1pmI1vDuuhNlj6rOcbjY9BWl1dcckiN65GFjMVw8CyUTRo5VBDTvWCcnZGPQVdRjr2xlR8QlDASDEddW?key=zXb3nihNyp9ChW3RfTALsL42" alt=""><figcaption></figcaption></figure>

First, users prove they own a valid identity document (passport or EU ID card) by generating a zero-knowledge proof of validity. This is done by proving the existence of a valid certificate authority chain for the user's document data. This proof is generated in a TEE rather than on the user's mobile phone for performance reasons. To ensure that the TEE doesn't save or leak the user's personal information, the mobile app will connect to the TEE only after it verifies the TEE attestation which can verify the code that is running on the TEE.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcd613h-jB86X20eiivmvD8aVQF6XheqW9bYz-k78x5s8Gg0xn_ws1NIoFT-7VWGsAX6HiKRaMs3p3d8NyXOcRpoG-uCuPuBWT1mFziq9-GOXSqDiklo9FRgin0Y6IqObSZg63IOQ?key=zXb3nihNyp9ChW3RfTALsL42" alt=""><figcaption></figcaption></figure>

The resulting zk proof is then verified onchain and their identity commitment is added to the identity registry. The registry is in the form of a merkle tree, whereas the identity commitment is a hash of several of the passport’s key information points, along with a user-generated secret. We call the registry the identity pool.

Subsequently, every time an application needs to check a user’s identity, the user can generate another zero-knowledge proof, the disclosure proof, that shows that they know how to reconstruct a commitment which is part of the identity pool. In particular, they need both the secret that was used when registering, and the passport data. In the proof, they are free to disclose information related to their identity using the DG1 information present in the commitment, while redacting any information they want to keep private.

\
For example, such disclosure proofs can prove that a user is over a certain age, that they are a citizen of a certain country, that they are not a citizen of a set of countries, and that they are not on the OFAC list. These statements can be linked to an user's wallet address in the proof, which can be shared with third party applications or onchain protocols. Disclosure proofs can be requested by third party applications using the Web SDK.

The goal of splitting registration and disclosure proving is two-fold:

* First, it creates a level of indirection between registration and disclosure. As detailed below in the Nullifier section, a country storing all attestations or an attacker stealing an attestation could identify users with their nullifiers. Thus, such an attacker could see which users have registered, but thanks to this level of indirection, they would not be able to see which actions the user has carried out (ie the user's disclosure proofs).
* Disclosure proofs are mainly Poseidon hashes, so they are cheap enough to be generated client side easily, whereas proving the passport validity involves much larger circuits.

As mentioned above, proofs are currently generated on Secure Enclaves. Additionally, all proofs are relayed onchain using our relayer so users don’t have to manage gas.

We’ll now detail some of the main design choices we made while building Self protocol.

### Nullifiers

Self uses two nullifiers: attestation nullifiers and action nullifiers. Attestation nullifiers prevent the creation of multiple identities with one passport. Action nullifiers prevent performing an action multiple times with the same identity.

#### Attestation Nullifiers

Attestation nullifiers are derived by hashing the passport’s signed attributes (signed\_attr in the circuits) using poseidon. Signed attributes is the final message signed by the DSC certificate. It incorporates enough entropy from previous datagroups like DG2 (the photo) so that it’s not vulnerable to glossary attacks. We used to hash the final signature instead of signed\_attr, but signing signed\_attr is simpler and prevents any kind of malleability attack on ECDSA signatures.

Because it’s derived deterministically from the passport’s passive attestation, an issuer keeping records of all signatures, or an attacker obtaining the passport’s attestation, can identify if a user registered. But thanks to the level of indirection between registration and disclosure, they can’t identify which actions the user took.

In such an attack, the user and the issuer/attacker have exactly the same information, so it’s not possible to distinguish them easily without adding additional mechanisms like trust relying on biometrics.

We explored designs such as keeping a salt in a TEE or a threshold secret using MPC and using an OPRF so salt attestation nullifiers. However, those solutions add assumptions on the absence of censorship, because losing access to this salt or preventing people from accessing the OPRF would prevent deduplication forever. This goes against our philosophy of keeping registration permissionless. We believe that knowing if the user registered is not very useful for attackers, as our goal is to host many applications, and given appropriate delays, the anonymity set can be large enough to guarantee privacy.

Another possible path would have been to use Active Authentication when available (detailed below) to derive deterministic nullifiers that an attacker couldn’t derive with the passport’s passive attestation. This would have been done by having the private key embedded in the passport to sign a fixed, universal message that would always give the same nullifier. However, even when the Active Auth signatures use RSA, a salt is always added by the chip to the challenge signed, because the challenge is only 8 bytes long. This is done to prevent an attacker from bypassing Active Auth by presigning all challenges. This means in practice, Active Auth cannot be used to generate such a nullifier.

#### Action Nullifiers

Action nullifiers are derived by hashing the user’s secret along with a scope using Poseidon. The scope is a unique identifier of the application requesting a proof (i.e. an airdrop, or a website). The scope will be generated deterministically from the DNS of the application requesting the disclosure proof. This information will be verified in the mobile app to prevent applications from extracting a nullifier that the user has already used.

### Commitment

During registration, the user’s commitment is added as a new leaf of the commitment merkle tree. This tree is deployed onchain and uses @zk-kit's[ lean-imt](https://github.com/privacy-scaling-explorations/zk-kit.solidity/tree/main/packages/lean-imt) library to ensure a scalable design.

The commitment is a hash of several passport related data. Some of them are not used in this first version of the protocol, but as we want to build this privacy pool over the time and let the commitment structure untouched, we already include them.

The values hashed to generate the commitment are:

* <mark style="color:green;">`secret`</mark> only known by users, brings entropy and is used to generate <mark style="color:green;">`action nullifiers`</mark>
* <mark style="color:green;">`dg1`</mark> the hash of the first DG, used for selective disclosure
* <mark style="color:green;">`eContent`</mark> contains the hash of all the DG present in the passport. If we want to use D15 for active authentication or DG2 for zkML face matching we’ll just have to unfold eContent to use the DGs.
* <mark style="color:green;">`DSC`</mark> certificate signing the passport. If one gets leaked, we will want to do a proof of non inclusion of a DSC hash.
* <mark style="color:green;">`CSCA`</mark> certificate signing the DSC, recorded for the same reason.

### DSC and CSCA trees

As mentioned above, verifying the validity of a passport involves two steps: checking it’s been signed by a DSC, and checking that the DSC has been signed by a CSCA. But proving those two steps for every registration would be expensive, and each DSC signs many passports. So every time we encounter a new DSC, we do a proof that it’s been signed by a CSCA and whitelist it by adding its hash to the DSC merkle tree. That way, at registration, users just have to prove that their passports has been signed by a certificate in the DSC tree.

#### CSCA tree

The CSCA tree is built from the ICAO masterlist registry using the scripts in[ /registry](https://github.com/zk-passport/openpassport/tree/main/registry). Each leaf is constructed as follows:

```
csca_leaf = poseidon2([csca_hash, csca_actual_length])
```

white <mark style="color:green;">`csca_hash`</mark> being the raw CSCA padded with zeros to 1792 bytes, packed and hashed with poseidon. We pad certificates with 0s so we have a common length for them, as dynamic length poseidon hashing is hard to implement in circom. We chose 1792 because the longest certificate present in the masterlist is <mark style="color:green;">`1591`</mark> bytes long. We have to commit to the actual size of the certificate so when doing a proof, it’s not possible to point to the position of the certificate public key in the zero padding.

Currently, the CSCA merkle tree root is managed by the owner account. We only update the root of the tree in a smart-contract, and give access to the whole tree offchain. The tree construction is fully auditable by running the scripts in[ /registry](https://github.com/zk-passport/openpassport/tree/main/registry). In the future, we can make sure the client rebuilds the CSCA tree to make sure it’s formed correctly, in the same way that we made proof verification transparent for the[ New American Primary](https://newdemocraticprimary.org/results). We also plan on decentralising the management of the CSCA tree in the future, most likely using multisigs or oracles.

The size of the CSCA tree is set to 12, which represents more than 4k leaves. There’s currently \~500 CSCA present in the ICAO Masterlist.

#### DSC tree

At registration, users prove that their passport has been signed by a DSC present in the DSC merkle tree. Each leaf of this tree is computed as the following:

```
dsc_leaf = poseidon2([
    poseidon2([dsc_hash, dsc_actual_length])
    poseidon2([csca_hash, csca_actual_length]) // this is also csca_leaf
])
```

Same as before, the <mark style="color:green;">`dsc_hash`</mark> is built by padding the raw DSC up to 1792 bytes. For technical reasons, the DSC is not just padded with 0s but with the sha padding, which includes a 128 bytes after the content and the actual size of the certificate after a certain number of blocks. This is because certificate signatures are checked with a dynamic sha in the <mark style="color:green;">`dsc.circom`</mark> circuit, so we pad the certificates outside the circuit.

Adding leafs to this tree is permissionless, meanings that anyone proving that a given DSC has been signed by a CSCA can add the corresponding leaf.

This tree is deployed onchain, and just like for the commitment merkle tree, we use <mark style="color:green;">`@zk-kit`</mark>'s[ lean-imt](https://github.com/privacy-scaling-explorations/zk-kit.solidity/tree/main/packages/lean-imt) library for scalability.

The reason we keep information about the CSCA in those leafs is so that they can be passed to the commitment in the registration proofs. That way, we can keep track in commitments of which CSCA was responsible for this commitment being added, and if a CSCA is compromised, potentially blacklist all commitments generated from it.

Each time a user scans their passport, the mobile application reads the DSC from the passport and checks its presence in the DSC merkle tree. If it’s not there, it sends the DSC to the TEE that will generate the DSC proof and relay it onchain.

We set the maximal depth of the DSC tree to 21 in the circuits which allows having more than 2M leaves. As the onchain tree is incremental, if this value is exceeded (unlikely in the next 10 years), we’ll have to build new circuits.

### Secret Management and Recovery

When registering an identity commitment, the user generates a secret. This prevents someone else that would access their passport from impersonating them, and brings the entropy necessary to generate application nullifiers that can’t be linked to passports. It also allows disclosure proofs to be done on the fly, instead of having to prove the whole passport validity every time.

Just like with wallets, it’s important that users don’t lose access to their secret, otherwise they can’t generate disclosure proofs. We prevent that with multiple mechanisms:

First, the secret is stored in the user’s keychain on their app. This allows the user to store it and retrieve it even if they uninstall and reinstall the app. Depending on whether the user uses iOS or Android, and has keychain cloud backups enabled, they can also retrieve it from other devices. When this is turned on by the user, this places a trust assumption on Apple/Google, that we think is acceptable for most users. We also let users store their secret themselves by showing them the BIP39 seed phrase associated with their secret, so that they can manage it themselves if they don’t trust Apple or Google.

Second, we prompt users to do an extra backup on cloud services, which are iCloud for iOS and Google Drive for Android. This is similar to what many wallets do, and provide an extra level of backup with the same trust assumptions mentioned above. Cloud backups are less sensitive than wallet seed phrases, because passport data is still necessary to use the identity, whereas with wallets funds can be spent directly.

Third, we designed the architecture so that it’s possible to add a recovery mechanism for identities. One central attack vector is that, while committing to a secret prevents someone else accessing the passport to do disclosure proofs, it also allows an attacker to register before the user if they get access to their passport. The way recovery could work is the following: the user initiates recovery by proving that they have a passport that corresponds to an existing nullifier, and after a delay, if the previous owner does not intervene, they get to replace the commitment corresponding to this nullifier with a new one with a new secret. This is not very practical with Passive Authentication, as passive attestations can be stored, but becomes way better when using Active Authentication, as being in physical possession of the passport is required, and a signature of a recent blockhash can be asked for, so it’s not possible for an attacker to access the passport once then initiate recovery over and over. More details on Active Authentication below.

In this last case, recovering an identity lets someone register a new commitment with a new secret while disabling the previous one. So that users can't use both commitments to do disclosure proofs, we can keep track of commitments that were disabled in a list or a sparse merkle tree, and require disclosure proofs to prove their commitment is not part of this tree. It’s possible to add that to our design before our contracts are upgradeable and it wouldn’t have to change the format of the commitment, although new circuits and trusted setups would be required.

Just like wallets do, we aim at being transparent in the app UI that backing up the secret is important, and that losing it entails not being able to do proofs with this identity document, at least before we ship a recovery mechanism for identities that lets people add commitment with a new secret.

### Trusted Setup

We ran the trusted setup ceremony on [ceremony.pse.dev](https://ceremony.pse.dev/projects/Self%20ZK%20Passport%20Ceremony). Thanks to the p0tion team at PSE for making this happen!

p0tion manages coordination between contributors by letting them connect with Github, get in a queue, and contribute to each circuit. It doesn’t add trust assumption, as contributors choose which hardware they want to use to contribute, and can add extra randomness to their toxic waste. In the past, we ran our own p0tion infra for the[ New Democratic Primary](https://newdemocraticprimary.org/results) we organized, but this time we got approval from PSE to run it on their p0tion infra, which simplifies things for us, for instance because it manages storing and distributing zkeys. p0tion also provides full auditability by publishing all intermediate zkeys and metadata regarding contributions, so anyone can verify the whole ceremony.

Our trusted setup seems to be one of the largest to be done with Groth16, with around 38 circuits ranging from 143k to 10.6M circom constraints. We will continue running setups when adding support for more countries.

### Zero-knowledge proof stack

We choose Circom with Groth16 for multiple reasons:

* Because it’s the oldest stack, it’s been examined carefully and has been used in production in a variety of applications. More recent stacks like Noir are easier to start with, but have not been audited and are not ready for production use.
* Fully succinct proofs with cheap onchain verification gas costs.
* An optimized proving stack with[ https://github.com/0xPolygonID/witnesscalc](https://github.com/0xPolygonID/witnesscalc) and[ https://github.com/iden3/rapidsnark](https://github.com/iden3/rapidsnark), along with tooling like[ https://github.com/privacy-scaling-explorations/p0tion](https://github.com/privacy-scaling-explorations/p0tion), which yields fast proving time.

The main downsides of Groth16 for us are the following:

* Circuit-specific trusted setups
* Large proving key

For the next iterations of Self, we’re excited by newer proving stacks that break those tradeoffs.

### Timing Attack prevention

The register and disclosure flow of the Self protocol resembles the Tornado Cash design. It functions as an identity mixer, and is susceptible to the same vulnerability known as a timing attack.

Here's how it works: A user registers by generating a commitment and adding it to the tree. If they generate their first disclosure proof and verify it onchain shortly after, linking the disclosure proof to their user identifier, an observer monitoring chain activity can reasonably assume it's the same person. If this observer has access to passport data, either because it’s the issuing country or because it accessed it in the past, they link the person’s identity to their identifier by deriving their nullifier. For onchain applications, the user identifier is most of the time their address.

There are two scenarios to consider here:

1. The disclosure proof is verified offchain by an application that does not publish proofs. In this case, the application would have to collude with an issuer or an attacker to link identities to their address.
2. The disclosure proof is verified onchain, or published by an application that verifies it offchain. In this case, if no delay has been introduced, it’s possible to look at which merkle root was used to do the disclosure proof, and infer that the person registered just before.

Our main mitigation is to communicate clearly with the user when they could be exposed to a timing attack. When they first register, they are prompted not to do a disclosure proof right away, but to wait until a delay has passed. We can balance security and user experience by sending a notification to users after a random time that invites them to generate their disclosure proof. This random time interval can be chosen according to the number of registration happening, so as to keep the anonymity set reasonable.

### Active Authentication

We currently use Passive Authentication, which involves verifying the attestation that issuers store on chips. More recently, new security mechanisms such as Active Authentication (AA) and Chip Authentication (CA) have been adopted by some countries. In both mechanisms, the passport’s chip is equipped with an embedded private key.

In AA, the passport signs an 8 bytes challenge provided by the reader. The public key of the passport can be found in DG15, and the integrity of DG15 can be verified with the passive attestation.

In CA, the passport and the reader perform DH key exchange and derive a symmetric key. The public key of the passport can be found in DG14.

[Support for AA and CA varies by countries.](https://www.inverid.com/blog/cloning-detection-identity-documents)

CA can’t be used to craft signatures because its all messages are repudiable, but AA can. Although we don’t support registration with Active Authentication currently, we designed Self so that it can be added easily in the future. The logic would be the following:

* In the registration circuit, check if DG15 is present.
  * If it is not, register with the passive attestation.
  * If it is, the circuit requires the prover to provide a valid signature of a fixed message that matches the public key in DG15.

To prevent issuers or attackers from pre-signing messages using AA when they have access to the passport, we can require the message to be a recent blockhash, for instance of the Ethereum blockchain. Assuming it’s infeasible to predict them and it’s impractical to extract private keys from secure chips, it would guarantee that the person registering is in physical possession of the passport.

\


<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXdQdcMtkAcMZZNEMANBY7vN-9Z26GXxesQ8VzgXhm-zwroGDcBzLMVRODhTGhQ_1LBVUF_YasngAVR_g0quhOr0zDuqw447B92XEx5uQwNmB_sijSMedSk-XFIKPsmcXlF7Pk0Lpg?key=zXb3nihNyp9ChW3RfTALsL42" alt="" width="563"><figcaption></figcaption></figure>

We are optimistic on the benefits of adding support for Active Authentication for the passports that support it, and we believe our architecture will requires little change to do so.

### Adding back client-side proving

Up until recently, we used to use only client-side proving, but we recently switched to using Secure Enclaves. This is due both to the compute requirements of proving for some signature algorithms such as ECDSA, and proving key size. However, we devised a way to add back enough client-side proving so that there would be no trust assumption on Secure Enclaves for privacy. We can still offload some of the computation to a server/trusted enclave by only leaking which DSC was used to sign the passport. The way we can it is the following:

* For registration proofs, we split the process between a small proof generated client side that just hashes DG1, and a larger proof generated server-side that includes the rest of the registration logic. Both proofs output a blinded commitment of DG1 that can be checked to make sure they refer to the same passport.
* For disclosure proofs, they are very light so they can be generated locally.

We estimate the maximal size of the proving key for this new registration circuit to be around 30mb zipped, which should be manageable for most devices, even with poor bandwidth.

\
\


