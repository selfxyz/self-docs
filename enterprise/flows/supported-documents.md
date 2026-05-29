# Supported documents

Each credential type supports a different subset of disclosures and a different geographic footprint. Pick documents based on the user populations you serve.

## Biometric passport (ICAO 9303 e-passport)

**Highest assurance.** A passport-chip read produces a credential signed by the issuing state.

* **Coverage:** 60+ countries. See [Supported countries](../reference/supported-countries.md).
* **Supports:** every disclosure, `age_gte` / `age_lte`, `nationality_in` / `nationality_not_in`, `ofac_clear`, `is_human`, `reveal_nationality`, `reveal_age_bucket`, `reveal_document_type`.
* **UX:** user holds passport to phone NFC reader. ~30 seconds end-to-end.
* **Notes:** requires a passport-chip-enabled phone (most iPhones and Android 8+).

## Aadhaar (India)

Government-issued universal ID for India.

* **Coverage:** India only.
* **Supports:** `age_gte` / `age_lte`, `is_human`, `reveal_age_bucket`, `reveal_document_type`. Does **not** support `nationality_*` (Aadhaar doesn't encode nationality the same way) or `ofac_clear` directly.
* **UX:** user scans the QR code on their Aadhaar card and provides their share code.
* **Notes:** see [Aadhaar spec](../reference/document-specifications/aadhaar.md) for the cryptographic detail.

## KYC attestation (partner-issued)

A credential issued by a Self-partner KYC provider after a remote KYC check.

* **Coverage:** depends on the issuer. Most partners cover US, EU, UK, and a long tail.
* **Supports:** disclosures depend on the issuer's schema. Typically `age_gte`, `nationality_in/not_in`, `ofac_clear`, `is_human`, `reveal_nationality`.
* **UX:** user completes a KYC flow once with the partner; subsequent verifications reuse the attestation.
* **Notes:** see [KYC spec](../reference/document-specifications/kyc.md) for the attestation format.

## Picking documents for your flow

* **Global consumer app:** biometric passport + KYC attestation. Maximizes coverage.
* **India-only product:** Aadhaar + biometric passport. Aadhaar penetration is much higher than passport ownership for the Indian population.
* **Compliance-heavy use case:** biometric passport only. Highest assurance, government-signed.
* **Reusable verification (user verifies once, you trust it for a year):** KYC attestation. The user doesn't re-scan a document each time.

## Capability matrix

| Disclosure | Biometric passport | Aadhaar | KYC attestation |
| --- | :---: | :---: | :---: |
| `age_gte` / `age_lte` | ✅ | ✅ | ✅ (issuer-dependent) |
| `nationality_in` / `nationality_not_in` | ✅ | ❌ | ✅ (issuer-dependent) |
| `ofac_clear` | ✅ | ❌ | ✅ (issuer-dependent) |
| `is_human` | ✅ (passport) | ✅ (Aadhaar) | ✅ (issuer-attested) |
| `reveal_nationality` | ✅ | ❌ | ✅ |
| `reveal_age_bucket` | ✅ | ✅ | ✅ |
| `reveal_document_type` | ✅ | ✅ | ✅ |

The dashboard's Configure tab flags incompatible combinations, if you allow only Aadhaar and add a `nationality_not_in` rule, you'll see a warning before publishing.

## Related

* [Anatomy of a flow](anatomy.md).
* [Disclosures](disclosures.md).
* [Supported countries](../reference/supported-countries.md).
* [Aadhaar spec](../reference/document-specifications/aadhaar.md).
* [KYC spec](../reference/document-specifications/kyc.md).
