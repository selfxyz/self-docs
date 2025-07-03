# Config Storage

## IConfigStorage Interface

The IConfigStorage interface is a key component of the Self Protocol V2 architecture that enables dynamic, context-aware verification configurations. It replaces the static configuration methods from V1 with a flexible storage pattern.

### Overview

IConfigStorage allows your application to:

* Store multiple verification configurations
* Select configurations dynamically based on user context
* Implement custom logic for configuration management
* Support A/B testing and gradual rollouts

### Interface Definition

```typescript
interface IConfigStorage {
  // Get a verification configuration by ID
  getConfig(configId: string): Promise<VerificationConfig>;
  
  // Store or update a configuration
  setConfig(configId: string, config: VerificationConfig): Promise<boolean>;
  
  // Determine which configuration to use based on context
  getActionId(userIdentifier: string, userDefinedData: string): Promise<string>;
}
```

#### VerificationConfig Type

```typescript
interface VerificationConfig {
  olderThan?: number;              // Minimum age requirement
  excludedCountries?: string[];    // ISO 3-letter country codes to exclude
  ofac?: boolean;                  // Enable OFAC sanctions checking
}
```

### Built-in Implementations

#### DefaultConfigStore

A simple implementation that uses a single configuration for all verifications:

```typescript
import { DefaultConfigStore } from '@selfxyz/core';

const configStore = new DefaultConfigStore({
  olderThan: 18,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
});
```

#### InMemoryConfigStore

A more flexible implementation that stores multiple configurations in memory:

```typescript
import { InMemoryConfigStore } from '@selfxyz/core';

const configStore = new InMemoryConfigStore(
  // Function to determine config ID from context
  async (userIdentifier: string, userDefinedData: string) => {
    const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
    return data.action === 'high_value' ? 'strict' : 'standard';
  }
);

// Set up different configurations
await configStore.setConfig('standard', {
  olderThan: 18,
  ofac: false
});

await configStore.setConfig('strict', {
  olderThan: 21,
  excludedCountries: ['IRN', 'PRK', 'CUB'],
  ofac: true
});
```

### Custom Implementations

#### Database-Backed Storage

```typescript
class DatabaseConfigStore implements IConfigStorage {
  constructor(private db: Database) {}
  
  async getConfig(configId: string): Promise<VerificationConfig> {
    const config = await this.db.query(
      'SELECT * FROM verification_configs WHERE id = ?',
      [configId]
    );
    
    if (!config) {
      throw new Error(`Config ${configId} not found`);
    }
    
    return {
      olderThan: config.minimum_age,
      excludedCountries: JSON.parse(config.excluded_countries),
      ofac: config.ofac_enabled
    };
  }
  
  async setConfig(configId: string, config: VerificationConfig): Promise<boolean> {
    await this.db.execute(
      `INSERT INTO verification_configs (id, minimum_age, excluded_countries, ofac_enabled) 
       VALUES (?, ?, ?, ?) 
       ON DUPLICATE KEY UPDATE 
       minimum_age = VALUES(minimum_age),
       excluded_countries = VALUES(excluded_countries),
       ofac_enabled = VALUES(ofac_enabled)`,
      [
        configId,
        config.olderThan || null,
        JSON.stringify(config.excludedCountries || []),
        config.ofac || false
      ]
    );
    return true;
  }
  
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    // Decode the user-defined data
    const decodedData = Buffer.from(userDefinedData, 'hex').toString();
    
    try {
      const data = JSON.parse(decodedData);
      
      // Custom logic based on your application needs
      if (data.action === 'withdraw' && data.amount > 10000) {
        return 'high_value_withdrawal';
      }
      
      if (data.action === 'create_account') {
        return 'new_user_onboarding';
      }
      
      return 'default_config';
    } catch {
      return 'default_config';
    }
  }
}
```

#### Redis-Backed Storage with Caching

```typescript
class RedisConfigStore implements IConfigStorage {
  constructor(
    private redis: RedisClient,
    private cacheTTL: number = 300 // 5 minutes
  ) {}
  
  async getConfig(configId: string): Promise<VerificationConfig> {
    const cached = await this.redis.get(`config:${configId}`);
    
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Fetch from primary storage if not cached
    const config = await this.fetchFromDatabase(configId);
    
    // Cache for future requests
    await this.redis.setex(
      `config:${configId}`,
      this.cacheTTL,
      JSON.stringify(config)
    );
    
    return config;
  }
  
  async setConfig(configId: string, config: VerificationConfig): Promise<boolean> {
    // Update primary storage
    await this.updateDatabase(configId, config);
    
    // Invalidate cache
    await this.redis.del(`config:${configId}`);
    
    return true;
  }
  
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    // Check for user-specific overrides
    const override = await this.redis.get(`user:${userIdentifier}:config`);
    if (override) {
      return override;
    }
    
    // Default logic
    const data = Buffer.from(userDefinedData, 'hex').toString();
    const parsed = JSON.parse(data);
    
    return parsed.configId || 'default';
  }
}
```

#### A/B Testing Implementation

```typescript
class ABTestConfigStore implements IConfigStorage {
  constructor(
    private defaultConfig: VerificationConfig,
    private testConfig: VerificationConfig,
    private testPercentage: number = 10 // 10% get test config
  ) {}
  
  async getConfig(configId: string): Promise<VerificationConfig> {
    return configId === 'test' ? this.testConfig : this.defaultConfig;
  }
  
  async setConfig(configId: string, config: VerificationConfig): Promise<boolean> {
    if (configId === 'test') {
      this.testConfig = config;
    } else {
      this.defaultConfig = config;
    }
    return true;
  }
  
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    // Use consistent hashing to ensure same user always gets same config
    const hash = crypto
      .createHash('sha256')
      .update(userIdentifier)
      .digest('hex');
    
    const hashValue = parseInt(hash.substring(0, 8), 16);
    const isTestGroup = (hashValue % 100) < this.testPercentage;
    
    return isTestGroup ? 'test' : 'default';
  }
}
```

### Integration with Frontend

The `userDefinedData` parameter in the frontend's SelfAppBuilder is passed to your `getActionId` method:

```javascript
// Frontend
const selfApp = new SelfAppBuilder({
  // ... other config
  version: 2,
  userDefinedData: "0x" + Buffer.from(JSON.stringify({
    action: "high_value_transaction",
    amount: 50000,
    merchant: "merchant_123"
  })).toString('hex'),
  // ...
}).build();

// Backend - getActionId receives this data
async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
  const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
  
  if (data.action === 'high_value_transaction' && data.amount > 10000) {
    return 'strict_verification';
  }
  
  return 'standard_verification';
}
```

### Best Practices

1. **Keep Configurations Consistent**: Ensure frontend disclosures match backend configurations
2. **Handle Errors Gracefully**: Always have a fallback configuration
3. **Cache Appropriately**: Balance performance with configuration freshness
4. **Version Your Configs**: Consider adding version fields for easier migrations
5. **Audit Configuration Changes**: Log all configuration updates for compliance
6. **Test Thoroughly**: Verify configuration selection logic with unit tests

### Migration from V1

Moving from V1's static configuration to V2's IConfigStorage:

```typescript
// V1 Pattern (No longer supported)
const verifier = new SelfBackendVerifier(scope, endpoint);
verifier.setMinimumAge(18);
verifier.excludeCountries('Iran', 'North Korea');
verifier.enableOfacCheck();

// V2 Pattern with IConfigStorage
const configStore = new DefaultConfigStore({
  olderThan: 18,
  excludedCountries: ['IRN', 'PRK'],
  ofac: true
});

const verifier = new SelfBackendVerifier(
  scope,
  endpoint,
  false,
  allowedIds,
  configStore,
  'uuid'
);
```

### Common Patterns

#### Multi-tenant Configuration

```typescript
class TenantConfigStore implements IConfigStorage {
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    const data = JSON.parse(Buffer.from(userDefinedData, 'hex').toString());
    return `tenant_${data.tenantId}_config`;
  }
  
  async getConfig(configId: string): Promise<VerificationConfig> {
    // Extract tenant ID and fetch their specific config
    const [, tenantId] = configId.match(/tenant_(\w+)_config/) || [];
    return this.fetchTenantConfig(tenantId);
  }
}
```

#### Time-based Configuration

```typescript
class TimeBasedConfigStore implements IConfigStorage {
  async getActionId(userIdentifier: string, userDefinedData: string): Promise<string> {
    const hour = new Date().getHours();
    
    // Stricter verification during night hours
    if (hour >= 22 || hour < 6) {
      return 'nighttime_strict';
    }
    
    return 'daytime_standard';
  }
}
```

This flexible configuration system enables sophisticated verification flows while maintaining security and compliance requirements.
