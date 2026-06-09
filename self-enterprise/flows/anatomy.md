# Anatomy of a flow

A **flow** is a published, versioned configuration of one product. It's what a `flowId` points to, and what you build in the dashboard's Configure tab.

## What a flow has

* **A product**: Pre KYC, Age Verification, or Proof of Human. The product decides what the user proves and which rules you can set.
* **A name** (plus a slug and an optional description): how you identify the flow.
* **Rules**: the predicate config (minimum age, excluded countries, OFAC), see below.
* From the Configure tab it also carries a **security level** (Standard or Hi-security), any **Additional data** reveals you turn on, and an **application icon**.

Two things are **not** part of a flow:

* **Document choice.** There's no per-flow document allowlist, the user verifies with whichever supported document they hold. See [Supported documents](supported-documents.md).
* **Redirect URLs.** `successUrl` and `failureUrl` are passed per session to `sessions.create(...)`, not stored on the flow.

## Rules

Rules are predicates over identity attributes. The user proves each one without revealing the underlying value. The published config is a small JSON object:

```json
{
  "minimumAge": 18,
  "excludedCountries": ["PRK", "USA"],
  "ofac": true
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `minimumAge` | integer | Age floor, 0 to 120. The user proves they are at least this old without revealing their birth date. |
| `excludedCountries` | string array | Issuing countries to deny. ISO 3166-1 alpha-3 codes, sorted. |
| `ofac` | boolean | When true, the user must clear the OFAC sanctions list. |

A Proof of Human flow has no rule fields; uniqueness is intrinsic to the document.

What the user's proof then attests to is a set of boolean outcomes, for example `{ "age_gte_18": true, "ofac_clear": true }`. See [Disclosures](disclosures.md).

## Immutable once published

A published flow **can't be edited**. Its config is frozen, captured as a version (`flowVersionId`), and every session pins to the exact version it ran against, so your audit log always shows what a user was verified against.

To change anything, you don't edit, you **archive the flow and create a new one** (with its own `flowId`). A product keeps **one active flow at a time**.

## Lifecycle

```
create + publish ──▶  active flow  ──archive──▶  archived (history kept)
                          │
                          └─ need a change? create a NEW flow (new flowId)
```

Archived flows stop accepting new sessions but keep their history for audit. See [Archive](../dashboard/overview.md#archive).

## Storage shape (for the curious)

If you integrate against the data model directly (for example via the dashboard's audit export):

```
flows
├─ id
├─ orgId
├─ product                   ← pre_kyc | age_verification | proof_of_human
├─ name
├─ slug
├─ description
├─ latestPublishedVersionId
└─ archivedAt
```

{% hint style="info" %}
On the wire the three products carry the internal slugs `pre_kyc`, `age_verification`, and `proof_of_human` (what the verifier expects). When creating sessions you reference a `flowId`, never the product slug.
{% endhint %}

## Related

* [Disclosures](disclosures.md): every outcome a proof can attest to.
* [Supported documents](supported-documents.md): what each credential type can prove.
* [Test vs. live](test-vs-live.md).
* [Configure a product](../dashboard/configure-a-product.md).
