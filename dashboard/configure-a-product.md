# Configure a product

Every flow is configured in the **Configure** tab of the product editor. Three sub-tabs.

> 📸 _**Screenshot:** Product editor with the Configure tab open, showing the three sub-tabs (Rules / Documents / Settings)._

## Rules

Rules are the predicates the user's ZK proof must satisfy. They map to disclosures: the user proves they meet the predicate without revealing the underlying attribute.

Common rules:

| Rule | Predicate | Notes |
| --- | --- | --- |
| Minimum age | `age >= N` | Doesn't reveal date of birth. |
| Nationality allow-list | `nationality in [...]` | Or `not in [...]`. ISO 3166-1 alpha-3. |
| OFAC | `not on OFAC list` | Sanctions check; uses Self's auto-updated list. |
| Proof of humanity | `is_human` | Always-on for Self Pass, passport-backed unique-human claim. |

Add a rule by clicking **Add rule** in the Rules sub-tab and selecting a predicate type. The dashboard validates the combination (e.g. you can't both allow-list and deny-list nationality on the same flow).

> 📸 _**Screenshot:** Rules sub-tab with a few sample rules added (age, nationality, OFAC)._

## Documents

Which credential types you'll accept. The dashboard shows everything supported; you tick the ones you want.

* **Biometric passport**: chip-equipped ICAO 9303 e-passports. Highest assurance.
* **Aadhaar**: Indian national ID. See [document spec](../reference/document-specifications/aadhaar.md).
* **KYC attestation**: a partner-issued KYC credential. See [KYC spec](../reference/document-specifications/kyc.md).

> **Coverage:** different documents have different country coverage. See [Supported countries](../reference/supported-countries.md).

## Settings

Per-flow operational settings:

* **Success URL**: where the user is sent after a successful verification.
* **Failure URL**: where the user is sent if the verification fails (wrong document, predicate not met, expired).
* **Display name**: what the user sees on the hosted page (e.g. "Acme Marketplace").
* **Branding**: logo + accent color for the hosted page.

These are referenced at session creation time. You can override `successUrl`/`failureUrl` per-session via the API for flows where the destination varies (e.g. deep links).

## Draft vs. published

Editing the Configure tab updates a **draft**. The draft is not live until you go to the [Deploy](publish-a-flow-version.md) tab and publish it.

In-flight sessions continue using the version they were created against, so publishing a new version never breaks an open session.

## Related

* [Anatomy of a flow](../flows/anatomy.md): the data model behind these tabs.
* [Disclosures](../flows/disclosures.md): what each rule actually proves.
* [Publish a flow version](publish-a-flow-version.md).
