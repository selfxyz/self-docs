# Disclosures

A disclosure is what the user's proof attests to. You configure them per flow in the [Configure](../dashboard/configure-a-product.md) tab, and which ones are available depends on the [product](../dashboard/configure-a-product.md#pick-a-product). There are two kinds:

* **Predicates**: a pass or fail answer (age, country, OFAC). The underlying value stays hidden.
* **Reveals**: the actual value of a document field, returned only when you ask for it.

## Security level

Choose how strictly the document is checked:

* **Standard**: the document is genuine.
* **Hi-security**: the user also physically scanned the document's chip.

## Age (predicate)

Available in **Pre KYC** and **Age Verification**. Set a minimum age; the user proves they meet it without revealing their birth date.

```json
{ "age_gte_18": true }
```

## Country (predicate)

Available in **Pre KYC**. Deny issuing countries with an excluded list (ISO 3166-1 alpha-3 codes). The user proves their document's country isn't on it, without revealing which country it is.

```json
{ "country_allowed": true }
```

## OFAC (predicate)

Available in **all products** (a toggle, on by default). The user proves they don't match the OFAC sanctions list, which Self keeps updated daily. On a match, the verification is rejected.

```json
{ "ofac_clear": true }
```

## Proof of human (predicate)

The core of the **Proof of Human** product, and present implicitly in the others since every verification is backed by a genuine document. The user proves they're a unique, real human via a nullifier derived from the document, stable per person in your product, unlinkable across products. See [How verification works](../get-started/how-it-works.md#one-account-per-person-nullifiers).

## Additional data (reveals)

Available in **Pre KYC**. Beyond the pass or fail predicates, you can ask the user to disclose specific document fields. Each is **off by default**, so request only what you need:

`Full name`, `ID number`, `Date issued`, `Date of birth`, `Gender`, `Nationality`, `Expiration date`, `Issuing state`.

A reveal returns the actual value, so it's personal data you then have to handle responsibly.

## What is never disclosed

* Anything you didn't configure. Predicates return only a pass or fail; reveals return only the fields you turned on.
* The user's biometric data and the raw document scan, those never leave the device.

## Related

* [Anatomy of a flow](anatomy.md).
* [Supported documents](supported-documents.md): which documents can satisfy which rules.
* [How verification works](../get-started/how-it-works.md): the model behind these predicates.
