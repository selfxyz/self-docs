---
icon: eye
---

# Disclosures

Disclosures control what information users reveal during identity verification. You configure them in the frontend `disclosures` object, which contains two types of settings:

1. **Verification Requirements** - conditions that must be met (must match backend)
2. **Disclosure Requests** - information users will reveal (frontend only)

## Verification Requirements

These settings define verification conditions and must match your backend `verification_config`:

### `minimumAge`
Verifies user is at least this age without revealing exact age or date of birth.

```javascript
disclosures: {
  minimumAge: 18, // User must be 18 or older
}
```

### `excludedCountries` 
Blocks users from specific countries using ISO 3-letter country codes.

```javascript
disclosures: {
  excludedCountries: ['IRN', 'PRK'], // Block Iran and North Korea
}
```

### `ofac`
Enables OFAC (sanctions) checking against official watchlists.

```javascript
disclosures: {
  ofac: true, // Enable sanctions checking
}
```

## Disclosure Requests

These settings specify what information users will reveal. Configure only in frontend - backend receives this data automatically.

### Personal Information

- **`name`**: User's full name from passport
- **`nationality`**: User's nationality 
- **`gender`**: User's gender (M/F)
- **`date_of_birth`**: Full date of birth

### Document Information

- **`passport_number`**: Passport number (use carefully for privacy)
- **`expiry_date`**: Passport expiry date
- **`issuing_state`**: Country that issued the passport

## Example Configuration

```javascript
disclosures: {
  // Verification requirements (must match backend)
  minimumAge: 21,
  excludedCountries: ['IRN'],
  ofac: true,
  
  // Disclosure requests (frontend only)
  nationality: true,
  gender: true,
  name: false,           // Don't request name
  date_of_birth: true,
  passport_number: false, // Don't request for privacy
}
```

## Verification Result

When verification succeeds, disclosed information is available in `result.discloseOutput`:

```javascript
const result = await selfBackendVerifier.verify(/*...*/);
if (result.isValidDetails.isValid) {
  const data = result.discloseOutput;
  
  console.log(data.nationality);    // "USA" (if requested)
  console.log(data.gender);         // "M" or "F" (if requested)
  console.log(data.olderThan);      // "18" (if minimumAge set)
  console.log(data.name);           // undefined (if not requested)
}
```

## Privacy Best Practices

- **Request only what you need**: Each disclosure reveals personal information
- **Avoid sensitive fields**: Be cautious with `passport_number` and `name`
- **Consider alternatives**: Use `minimumAge` instead of `date_of_birth` for age verification
- **Store carefully**: Implement proper data protection for disclosed information

## Common Use Cases

**Age verification only:**
```javascript
disclosures: {
  minimumAge: 18, // No personal data revealed
}
```

**Basic identity with nationality:**
```javascript
disclosures: {
  minimumAge: 18,
  nationality: true,
  gender: true,
}
```

**Complete identity verification:**
```javascript
disclosures: {
  minimumAge: 21,
  ofac: true,
  nationality: true,
  name: true,
  date_of_birth: true,
}
```