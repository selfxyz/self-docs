# User context data

## User Context Data

User Context Data is a hex-encoded string that carries contextual information from the frontend to the backend during the Self Protocol verification flow. It's a key component of V2 that enables dynamic verification configurations.

### Structure

User Context Data is a 256-byte hex string containing:

* **First 32 bytes**: Reserved for protocol use
* **Next 32 bytes**: User identifier (UUID or hex address)
* **Last 64 bytes**: User-defined data from frontend

```
0x[reserved_32_bytes][user_identifier_32_bytes][user_defined_data_64_bytes]
```

### Components

#### User Identifier (32 bytes)

The user identifier portion depends on the `userIdentifierType`:

```typescript
// For UUID type
const userId = uuidv4(); // "123e4567-e89b-12d3-a456-426614174000"
// Encoded as 32 bytes in the context data

// For hex type (blockchain addresses)
const userId = "0x742d35Cc6634C0532925a3b844Bc9e7595f1234";
// Padded to 32 bytes in the context data
```

#### User Defined Data (64 bytes)

This is the custom data you specify in the frontend:

```javascript
// Frontend: Creating user defined data
const actionData = {
  action: "withdraw",
  amount: 10000,
  sessionId: "xyz123"
};

const userDefinedData = "0x" + Buffer.from(
  JSON.stringify(actionData)
).toString('hex').padEnd(128, '0'); // 128 hex chars = 64 bytes
```

### Frontend Usage

Set user-defined data when building the Self app configuration:

```javascript
const selfApp = new SelfAppBuilder({
  appName: "My App",
  scope: "my-app",
  endpoint: "https://api.myapp.com/verify",
  userId: uuidv4(),
  version: 2,
  userDefinedData: "0x" + Buffer.from(JSON.stringify({
    action: "create_account",
    referralCode: "SUMMER2024",
    tier: "premium"
  })).toString('hex').slice(0, 128), // Ensure exactly 64 bytes
  disclosures: { /* ... */ }
}).build();
```

### Backend Usage

The backend receives the full user context data and can extract components:

```typescript
class ConfigStorage implements IConfigStorage {
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    // userIdentifier is already extracted for you
    console.log('User ID:', userIdentifier);
    
    // Decode the user-defined portion
    const decoded = Buffer.from(userDefinedData, 'hex').toString();
    
    try {
      const data = JSON.parse(decoded);
      
      // Use the data to determine configuration
      if (data.action === 'withdraw' && data.amount > 10000) {
        return 'high_value_config';
      }
      
      return 'standard_config';
    } catch {
      // Handle non-JSON data
      return 'default_config';
    }
  }
}
```

### Common Patterns

#### Action-Based Configuration

```javascript
// Frontend
userDefinedData: "0x" + Buffer.from(JSON.stringify({
  action: "transfer",
  recipient: "0x123...",
  amount: 50000
})).toString('hex').padEnd(128, '0')

// Backend
async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
  const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
  
  switch (data.action) {
    case 'transfer':
      return data.amount > 10000 ? 'transfer_high' : 'transfer_low';
    case 'login':
      return 'login_config';
    case 'register':
      return 'registration_config';
    default:
      return 'default_config';
  }
}
```

#### Session-Based Configuration

```javascript
// Frontend - Include session information
userDefinedData: "0x" + Buffer.from(JSON.stringify({
  sessionId: sessionStorage.getItem('sessionId'),
  timestamp: Date.now(),
  deviceType: navigator.userAgent.includes('Mobile') ? 'mobile' : 'desktop'
})).toString('hex').padEnd(128, '0')

// Backend - Use session data for configuration
async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
  const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
  
  // Check if this is a returning session
  const existingSession = await this.sessionStore.get(data.sessionId);
  
  if (existingSession) {
    return existingSession.isPremium ? 'premium_config' : 'standard_config';
  }
  
  // New session - check device type
  return data.deviceType === 'mobile' ? 'mobile_config' : 'desktop_config';
}
```

#### Multi-Tenant Configuration

```javascript
// Frontend - Include tenant information
userDefinedData: "0x" + Buffer.from(JSON.stringify({
  tenantId: window.location.hostname.split('.')[0], // subdomain
  environment: process.env.REACT_APP_ENV,
  feature_flags: ['new_flow', 'enhanced_checks']
})).toString('hex').padEnd(128, '0')

// Backend - Tenant-specific configuration
async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
  const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
  
  // Load tenant-specific configuration
  const tenantConfig = await this.getTenantConfig(data.tenantId);
  
  if (data.feature_flags.includes('enhanced_checks')) {
    return `${data.tenantId}_enhanced`;
  }
  
  return `${data.tenantId}_${data.environment}`;
}
```

### Size Limitations

The user-defined portion is limited to 64 bytes (128 hex characters):

```javascript
// ❌ Too large - will be truncated
const largeData = {
  action: "complex_operation",
  metadata: {
    field1: "value1",
    field2: "value2",
    // ... many more fields
  }
};

// ✅ Compact representation
const compactData = {
  a: "transfer",  // Use short keys
  v: 10000,       // Numeric values are efficient
  t: Date.now()
};

// For larger data, use references
const referenceData = {
  sessionId: "abc123",  // Store full data server-side
  version: 2
};
```

### Binary Encoding

For maximum efficiency, you can use binary encoding:

```javascript
// Encode multiple values in binary format
function encodeUserData(action: number, amount: number, flags: number): string {
  const buffer = Buffer.alloc(64);
  
  buffer.writeUInt8(action, 0);        // 1 byte for action
  buffer.writeBigUInt64BE(BigInt(amount), 1);  // 8 bytes for amount
  buffer.writeUInt32BE(flags, 9);      // 4 bytes for flags
  
  return "0x" + buffer.toString('hex');
}

// Frontend
userDefinedData: encodeUserData(
  1,      // Action: 1 = transfer
  50000,  // Amount: $50,000
  0b1101  // Flags: enhanced_checks | ofac | fast_track
)

// Backend
function decodeUserData(hex: string): { action: number, amount: number, flags: number } {
  const buffer = Buffer.from(hex, 'hex');
  
  return {
    action: buffer.readUInt8(0),
    amount: Number(buffer.readBigUInt64BE(1)),
    flags: buffer.readUInt32BE(9)
  };
}
```

### Security Considerations

1. **Don't include sensitive data**: User context data is not encrypted
2. **Validate all inputs**: Always validate decoded data
3. **Use checksums**: For critical data, include integrity checks
4. **Avoid user-controlled configs**: Don't let users directly specify security settings

```javascript
// ❌ Bad: User controls security settings
userDefinedData: "0x" + Buffer.from(JSON.stringify({
  skipOFAC: true,  // Don't allow this
  minimumAge: 0    // Don't allow this
})).toString('hex')

// ✅ Good: User provides context, backend decides security
userDefinedData: "0x" + Buffer.from(JSON.stringify({
  purchaseType: "alcohol",  // Backend will enforce age 21
  jurisdiction: "US-CA"     // Backend will apply CA laws
})).toString('hex')
```

### Integration with Smart Contracts

In V2 contracts, user context data flows through to the callback:

```solidity
function customVerificationHook(
    PassportData memory passportData,
    VerificationOutput memory output,
    uint256 attestationId,
    bytes32 userIdentifier,
    bytes calldata userDefinedData  // Your 64 bytes
) internal override {
    // Decode and use the data
    uint8 action = uint8(userDefinedData[0]);
    uint256 amount = uint256(bytes32(userDefinedData[1:33]));
    
    if (action == 1 && amount > 10000) {
        // High-value transfer logic
    }
}
```

### Debugging

To debug user context data issues:

```typescript
// Backend: Log the full context
console.log('Full context data:', userContextData);
console.log('User identifier:', userIdentifier);
console.log('User defined (hex):', userDefinedData);
console.log('User defined (decoded):', Buffer.from(userDefinedData, 'hex').toString());

// Check data size
console.log('User defined size:', Buffer.from(userDefinedData, 'hex').length, 'bytes');
```

### Best Practices

1. **Keep data compact**: Use short keys and efficient encoding
2. **Version your format**: Include a version field for future compatibility
3. **Document your schema**: Clearly document what data is expected
4. **Handle errors gracefully**: Always have fallback configurations
5. **Use standard formats**: Prefer JSON for readability, binary for efficiency
6. **Test edge cases**: Test with maximum size data and empty data
7. **Monitor usage**: Log what configurations are being selected

This flexible system enables sophisticated verification flows while maintaining security and performance.
