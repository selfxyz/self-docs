# Disclosure Configs

The disclosure configuration in `SelfAppBuilder` controls what verification rules are applied and what data users reveal during identity verification.

For the full reference including all available fields, verification rules, data disclosures, code examples, and privacy best practices, see the [Disclosures](../disclosures.md) page.

## Quick Reference

```typescript
const app = new SelfAppBuilder({
  // ... other config
  disclosures: {
    // Verification rules (must match backend/contract config exactly)
    minimumAge: 18,
    excludedCountries: ["IRN", "PRK"],
    ofac: true,

    // Data disclosures (frontend only — what users reveal)
    nationality: true,
    gender: true,
    name: false,
    date_of_birth: false,
    passport_number: false,
    expiry_date: false,
    issuing_state: false,
  },
}).build();
```

{% hint style="warning" %}
Verification rules (`minimumAge`, `excludedCountries`, `ofac`) **must exactly match** your backend or smart contract configuration. A mismatch will cause verification to fail. See [Frontend & Backend Alignment](../disclosures.md#frontend-and-backend-alignment) for details.
{% endhint %}
