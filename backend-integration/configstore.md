# ConfigStore

The **config storage layer** defines how verification configurations are persisted and retrieved on the backend. These configurations represent the same disclosure/verification rules that the frontend and smart contracts enforce. Keeping them consistent is critical.

This `@selfxyz/core` library exposes an interface (`IConfigStorage`) and two default implementations (`DefaultConfigStore`, `InMemoryConfigStore`).

## `IConfigStorage`&#x20;

```typescript
export interface IConfigStorage {
    getConfig(id: string): Promise<VerificationConfig>
    setConfig(id: string, config: VerificationConfig): Promise<boolean>
    getActionId(userIdentifier: string, data: string): Promise<string>
}
```

#### Methods

* **`getConfig(id)`**
  * Returns the `VerificationConfig` associated with a given id.
  * This id is typically the `configId` referenced by your contract/backend/frontend.
* **`setConfig(id, config)`**
  * Stores a new verification config under the given id.
  * Returns `true` if an existing config was replaced, `false` if it was newly set.
* **`getActionId(userIdentifier, data)`**
  * Computes or retrieves an action id based on user context data from the frontend.
  * This action id links user activity with a registered config and is later used by `getConfig(id)` &#x20;

## DefaultConfigStore

A simple implementation that always returns the same config regardless of id.&#x20;

```typescript
export class DefaultConfigStore implements IConfigStorage {
    constructor(private config: VerificationConfig) {}
    
    //other methods
}
```

### Example

```typescript
const defaultConfigStore = new DefaultConfigStore({
  minimumAge: 18,
  excludedCountries: ['US', 'CA'],
  ofac: true,
});
```

## InMemoryConfigStore

A more flexible implementation that can hold multiple configs in memory.

```typescript
export class InMemoryConfigStore implements IConfigStorage {
  private configs: Map<string, VerificationConfig> = new Map();
  private getActionIdFunc: IConfigStorage['getActionId'];

  constructor(getActionIdFunc: IConfigStorage['getActionId']) {
    this.getActionIdFunc = getActionIdFunc;
  }
  
  //other methods
}
```

### Example&#x20;

```typescript
const inMemoryHandler = async (userIdentifier: string, userDefinedData: string) => {
  return userDefinedData === 'high_value' ? 'strict' : 'standard';
};

const inMemory = new InMemoryConfigStore(inMemoryHandler);
inMemory.setConfig('strict', {
  minimumAge: 18,
  excludedCountries: ['USA', 'CAN'],
  ofac: true,
});
inMemory.setConfig('standard', {
  minimumAge: 18,
  excludedCountries: ['USA', 'CAN'],
  ofac: true,
});
```

**Use cases:**

* Backends that need to manage multiple verification configurations at once.
* Scenarios where action ids must be computed dynamically per user.
* Good for prototyping before connecting to a persistent store (DB, KV, etc.).

## Custom Implementations

Here's an example of a `KVConfigStore` that is used in the [playground](https://plaground.self.xyz/).

```typescript
import {
  IConfigStorage,
  VerificationConfig,
} from "@selfxyz/core";
import { Redis } from "@upstash/redis";

export class KVConfigStore implements IConfigStorage {
  private redis: Redis;

  constructor(url: string, token: string) {
    this.redis = new Redis({
      url: url,
      token: token,
    });
  }

  async getActionId(userIdentifier: string, data: string): Promise<string> {
    return userIdentifier;
  }

  async setConfig(id: string, config: VerificationConfig): Promise<boolean> {
    await this.redis.set(id, JSON.stringify(config));
    return true;
  }

  async getConfig(id: string): Promise<VerificationConfig> {
    const config = (await this.redis.get(id)) as VerificationConfig;
    return config;
  }
}
```

### Integration with Frontend

The `userDefinedData` parameter in the frontend's `SelfAppBuilder` is passed to your `getActionId` method:

```javascript
// Frontend
const selfApp = new SelfAppBuilder({
  // ... other config
  version: 2,
  userDefinedData: Buffer.from(JSON.stringify({
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

### Summary

* `IConfigStorage` defines a contract for working with verification configs.
* The library ships with `DefaultConfigStore` (static) and `InMemoryConfigStore` (multi‑config, dynamic).
* Backend verifiers (like `SelfBackendVerifier`) depend on `IConfigStorage` to fetch configs and action ids.
* You can plug in your own storage backend by implementing the same interface.
