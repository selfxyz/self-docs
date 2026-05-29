# Anatomy of a flow

A **flow** is a published, versioned configuration for a product. This page is the data model behind the dashboard's Configure tab.

## Three layers

```
Flow
├─ Rules        what the user must prove
├─ Documents    which credentials they can prove it with
└─ Settings     operational metadata (URLs, branding)
```

Each layer is independently configurable. They combine into a **flow version** when you publish.

## Rules

Predicates over identity attributes. The user proves they satisfy each predicate without revealing the underlying values.

Example rule set:

```json
{
  "age_gte": 18,
  "nationality_not_in": ["US", "CN", "IR"],
  "ofac_clear": true
}
```

The user's ZK proof attests to:

* `age_gte_18 = true` (they're at least 18, but you don't see their DOB)
* `nationality_not_in_US_CN_IR = true` (they're not from those countries, but you don't see which they are)
* `ofac_clear = true` (their identity isn't on the OFAC list)

See [Disclosures](disclosures.md) for the full catalog of rules.

## Documents

Which credential types satisfy this flow. The user picks from the documents you allow:

* `biometric_passport`: ICAO 9303 e-passports. 60+ countries.
* `aadhaar`: Indian national ID. See [Aadhaar spec](../reference/document-specifications/aadhaar.md).
* `kyc_attestation`: partner-issued KYC. See [KYC spec](../reference/document-specifications/kyc.md).

Documents map to predicate capability. A KYC attestation might not support `age_gte` if the issuer didn't include date-of-birth, the dashboard warns you when a rule isn't satisfiable by an allowed document.

## Settings

* **`displayName`**: what the user sees on the hosted page (e.g. "Acme Marketplace verification").
* **`successUrl`**: redirect on success.
* **`failureUrl`**: redirect on failure.
* **`branding`**: logo URL + accent color.

Per-session overrides for `successUrl` and `failureUrl` are accepted by the API, useful when the destination varies per user.

## Versions

A **flow** is the identity (the `flowId` you reference). A **flow version** is a frozen snapshot of `(rules, documents, settings)`. Every time you publish, a new version is created with a new `flowVersionId`.

* A session is pinned to a `flowVersionId` at creation time, it cannot move to a newer version mid-flight.
* The flow's `latestPublishedVersionId` is what new sessions use.
* Older versions are retained for audit and rollback.

## Lifecycle

```
draft ──publish──▶ published v1 ──edit──▶ draft ──publish──▶ published v2
                       │                                          │
                       └──── sessions still pinned to v1 ─────────┘
```

There's no explicit "archive a version". Archive happens at the flow level (`archivedAt`). An archived flow stops accepting new sessions but its history is intact.

## Storage shape (for the curious)

If you're integrating against the data model directly (e.g. via the dashboard's audit export):

```
flows
├─ id
├─ orgId
├─ productId
├─ name
├─ latestPublishedVersionId  ← FK to flow_versions
└─ archivedAt

flow_versions
├─ id
├─ flowId
├─ versionNumber
├─ configId                  ← FK to config (the raw predicate JSON)
└─ publishedAt
```

The `config` table holds the canonical predicate JSON. That's what the dashboard surfaces in the Configure tab.

## Related

* [Disclosures](disclosures.md): every available rule.
* [Supported documents](supported-documents.md): what each credential type can prove.
* [Test vs. live](test-vs-live.md).
* [Publish a flow version](../dashboard/publish-a-flow-version.md).
