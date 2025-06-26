---
description: An in-depth overview of the Self protocol.
---

# Overview

### Introduction

The Self protocol is an identity protocol designed to let people use their real-world attestation in a permissionless way, for Sybil resistance and selective disclosure. Our core thesis is that web-of-trust systems are hard to scale securely, and biometric verification à la Worldcoin has a long way to go, so bootstrapping from existing sources of trust like institutions is the most pragmatic way to verify identities securely today and in a privacy preserving way. We’re starting with passports, and national IDs.

Self has **three** main components:

* A mobile app that lets users easily scan the NFC chip in their passport. The mobile app operates both on iOS and Android, and performs the authentication mechanisms required by the passport to read the content of its chip (BAC/PACE).
* Zero-knowledge circuits that can be used to verify the validity of certificates and passports, generate identity commitments and selectively disclose attributes.
* Smart contracts that verify proofs, manage a merkle tree of identity commitments and allow for onchain disclosure of data while guaranteeing the permissionless aspect of the protocol.

### Background on Biometric Passports

Biometric passports were introduced in the 2000s as a way to streamline border control and reduce the risk of passport forgery. They are now issued in more than 170 countries, and their specifications are established by the ICAO (International Civil Aviation Organisation) and made available in Document 9303[ on their website](https://www.icao.int/publications/pages/publication.aspx?docnum=9303).

Each biometric passport contains an embedded microchip that can be read by any NFC reader. It stores multiple datagroups (up to 16) along with a SOD (Document Security Object) that can be used to verify the integrity of the passport. The SOD contains hashes of all datagroups, a signature attesting to the validity of the passport, information on which hash functions and signature algorithms were used, and the certificate that signed the passport.

All possible datagroups and their content can be found in the image below. DG1 and DG2 are mandatory, the rest is optional.\


<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXe4cgL2Q2UTAEroxDDJvjRn236hS6Cco-RDKcfIVbirrIuP7U2xx03gW_c-D6YINP5xWHD9_ktPTurewHPFMBLRmovwtiGBXasgpuHoqTUpnR1R4GDLgFF8yE-97refxe91Xdxu?key=zXb3nihNyp9ChW3RfTALsL42" alt=""><figcaption></figcaption></figure>

In particular:

* <mark style="color:green;">`DG1`</mark> has the same content as the machine readable zone, and is the source of all the information we care to verify.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfz3oE4332kG4YA0NfK4Svdz-Z1vLsZ-jIoJ7l1Fhzi_cSMdTuSpRQACair9_EG4VIBijrlhn4_seQdLyE8yV5SKfHZBs93Yl5H7FhB61ZT9xU3tNLwnZNN6GiPXs7ADQ8YarH7bA?key=zXb3nihNyp9ChW3RfTALsL42" alt=""><figcaption></figcaption></figure>

* <mark style="color:green;">`DG2`</mark> contains the person’s photo. Because it contains a lot of entropy, it makes sure that the final signature can’t be dictionary-attacked starting with some of the person’s information.
* <mark style="color:green;">`DG15`</mark> (optional) is the public key corresponding to the passport’s Active Authentication private key. We plan to use it to improve security in the future.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfSyDl3ZNPMRj8tWolIwlDKp9JhSId0Wf42vsEfA3Z5LJp7jWoo5CvXgpt6tPIdiYARyl4ALIx8CCdMTLhZlJEVQQuzNSdzp0JtKH-_7S4JazjMFUTd4dBmz0SUp4OABH86XH4uKw?key=zXb3nihNyp9ChW3RfTALsL42" alt="" width="188"><figcaption></figcaption></figure>

The passport data groups are hashed and process, and the resulting final hash is signed by a Document Signing Certificate (DSC), which is itself signed by a Country Signing Certificate Authority (CSCA) as part of a certificate chain. The DSC can be read from the passport's chip and the CSCA can be looked up on international registries such as the ICAO masterlist.

The <mark style="color:green;">`eContent`</mark> consists of the concatenation of all DG hashes, while the <mark style="color:green;">`signedAttr`</mark> is the final message signed by the issuing country. Sometimes, different hashing algorithms are used at each step.

According to the specifications, each DSC should sign up to 100k passports, each CSCA should rotate every 3-5 years, and a country should always have at minimum 2 valid CSCA at the same time.\
