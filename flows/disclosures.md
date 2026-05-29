# Disclosures

A disclosure is what the user's ZK proof attests to. There are two flavors: **predicates** (boolean, pass / fail) and **reveals** (the actual value, redacted to a category).

You configure these per flow under [Configure → Rules](../dashboard/configure-a-product.md#rules).

## Predicates

The user proves the predicate is true without revealing the underlying attribute.

### `age_gte` / `age_lte`, Age bounds

```json
{ "age_gte": 18 }
```

User proves they're at least 18. You don't see their date of birth or their exact age.

`age_lte` is also available (e.g. for a "must be under 25" flow). Combine `age_gte` and `age_lte` for a band.

`proof_attributes` output:

```json
{ "age_gte_18": true }
```

### `nationality_in` / `nationality_not_in`, Nationality

```json
{ "nationality_not_in": ["US", "CN", "IR", "KP"] }
```

User proves their issuing country is (or isn't) in the list. Use ISO 3166-1 alpha-3.

You can use `nationality_in` OR `nationality_not_in` per flow, not both.

`proof_attributes` output:

```json
{ "nationality_not_in_US_CN_IR_KP": true }
```

### `ofac_clear`, Sanctions

```json
{ "ofac_clear": true }
```

User proves their name + DOB don't match the current OFAC SDN list. We auto-update the list daily.

`proof_attributes` output:

```json
{ "ofac_clear": true }
```

If the user matches, the verification status is `invalid` and `ofac_clear` is omitted.

### `is_human`, Proof of humanity

```json
{ "is_human": true }
```

Always-on for Self Pass, a verified passport implicitly proves a unique human. You don't need to add this rule explicitly; it's enforced.

The uniqueness check uses a nullifier derived from the passport, so each real-world identity can only verify a flow once per session.

## Reveals

A reveal returns the actual value, not just a boolean. Use sparingly, every reveal is data your service then has to handle responsibly.

### `reveal_nationality`

```json
{ "reveal_nationality": true }
```

Returns the user's nationality (ISO 3166-1 alpha-3).

`proof_attributes` output:

```json
{ "nationality": "IND" }
```

### `reveal_age_bucket`

```json
{ "reveal_age_bucket": ["18-25", "26-35", "36-50", "51-65", "65+"] }
```

Returns which bucket the user falls into. Less revealing than exact age, more useful than a single threshold.

`proof_attributes` output:

```json
{ "age_bucket": "26-35" }
```

### `reveal_document_type`

```json
{ "reveal_document_type": true }
```

Returns whether the user verified with `biometric_passport`, `aadhaar`, or `kyc_attestation`.

## Combining disclosures

You can mix predicates and reveals freely. A realistic compliance flow might look like:

```json
{
  "age_gte": 18,
  "nationality_not_in": ["US"],
  "ofac_clear": true,
  "reveal_nationality": true
}
```

This gives you: a guaranteed-adult, non-US, sanctions-clear user, with their actual nationality (e.g. for tax determination).

## What's never disclosed

* The user's name.
* Their date of birth (only the `age_gte`/`age_lte` boolean or the age bucket).
* Their passport number.
* The biometric data.
* Their exact address (only nationality, if you ask).

The ZK proof attests to what the rules ask. Nothing else leaks.

## Related

* [Anatomy of a flow](anatomy.md).
* [Supported documents](supported-documents.md): which docs support which disclosures.
