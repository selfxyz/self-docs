# OFAC & CSCA Auto-Updaters

Self Pass relies on two automated systems to keep verification data current: the **OFAC sanctions list updater** and the **CSCA certificate tree updater**. These run automatically and require no action from developers integrating Self Pass.

## OFAC Sanctions List Updater

The OFAC (Office of Foreign Assets Control) auto-updater keeps the on-chain sanctions Merkle tree synchronized with the latest US Treasury OFAC sanctions data. This enables Self Pass to perform real-time sanctions screening as part of the zero-knowledge proof generation process.

**How it works:**

1. The updater fetches the latest OFAC sanctions data from the Self API
2. Three separate Merkle trees are maintained for different matching strategies:
   * **Passport number + nationality** — exact document match
   * **Name + date of birth** — biographical match
   * **Name + year of birth** — broader biographical match
3. The updated Merkle roots are submitted on-chain to the registry contract
4. When users generate proofs with OFAC enabled, the circuit verifies non-inclusion in these trees

ID card verification uses a subset of these trees (skipping the passport-number tree).

## CSCA Certificate Tree Updater

The CSCA (Country Signing Certificate Authority) auto-updater maintains the Merkle tree of trusted certificate authority public keys used to validate passport signatures.

**How it works:**

1. The updater fetches the latest certificates from the ICAO (International Civil Aviation Organization) masterlist
2. Public keys are extracted and validated against known Subject Key Identifiers (SKIs)
3. A Merkle tree is constructed from the validated public keys (~500 CSCAs, tree depth 12)
4. The updated root is submitted on-chain

This ensures that newly issued or rotated country signing certificates (which typically cycle every 3-5 years) are recognized by the verification system. The tree supports permissionless leaf addition, meaning new certificates can be added by anyone.

## For Developers

These systems are transparent to developers integrating Self Pass. When you enable OFAC checking in your disclosure configuration (`ofac: true`), the proof generation and verification automatically use the latest sanctions data. Similarly, passport signature verification always uses the most current CSCA tree.

The updater scripts live in the [Self Protocol contracts repository](https://github.com/selfxyz/self).
