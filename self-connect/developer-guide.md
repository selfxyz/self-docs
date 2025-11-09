# Developer Guide

### Getting Started

#### Installation and Setup

Install the required packages for Self Connect integration:

```bash
npm install @celo/identity @celo/abis viem
```

#### Required Packages

| Package          | Purpose                                             |
| ---------------- | --------------------------------------------------- |
| `@celo/identity` | ODIS integration and identifier utilities           |
| `@celo/abis`     | Contract ABIs for FederatedAttestations             |
| `viem`           | Modern Ethereum library for blockchain interactions |

#### Why Viem?

Self Connect uses **Viem** as the recommended library for blockchain interactions because:

* **Modern & Type-Safe**: Built with TypeScript for excellent type inference
* **Lightweight**: Smaller bundle size compared to legacy libraries
* **Modular**: Import only what you need
* **Better DX**: Cleaner API and better error messages
* **Active Maintenance**: Well-maintained with regular updates
* **EIP-1193 Support**: Native support for wallet providers

> **Note:** ContractKit is deprecated and no longer recommended for new projects.

#### Network Configuration

**Mainnet:**

```typescript
import { celo } from "viem/chains";

const RPC_URL = "https://forno.celo.org";
const ODIS_CONTEXT = OdisContextName.MAINNET;
```

**Alfajores Testnet:**

```typescript
import { celoAlfajores } from "viem/chains";

const RPC_URL = "https://alfajores-forno.celo-testnet.org";
const ODIS_CONTEXT = OdisContextName.ALFAJORES;
```

### Becoming an Issuer

#### Issuer Setup

An issuer needs:

1. A funded wallet account
2. ODIS quota for identifier obfuscation
3. Verification infrastructure
4. Optional: Data Encryption Key (DEK) for enhanced authentication

#### Step 1: Set Up Issuer Account

```typescript
import { createWalletClient, http, parseEther } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { celoAlfajores } from "viem/chains";

// Issuer private key - KEEP SECURE
const ISSUER_PRIVATE_KEY = process.env.ISSUER_PRIVATE_KEY;
const account = privateKeyToAccount(ISSUER_PRIVATE_KEY);

// Create Viem client
const walletClient = createWalletClient({
  account,
  transport: http(),
  chain: celoAlfajores
});

const issuerAddress = account.address;
console.log("Issuer Address:", issuerAddress);
```

#### Step 2: Authentication Methods

Self Connect supports multiple authentication methods for ODIS:

**Wallet Key Authentication**

```typescript
import { OdisUtils } from "@celo/identity";
import { AuthSigner } from "@celo/identity/lib/odis/query";

const authSigner: AuthSigner = {
  authenticationMethod: OdisUtils.Query.AuthenticationMethod.WALLET_KEY,
  sign191: ({ message, account }) => walletClient.signMessage({ 
    message, 
    account 
  })
};
```

**Encryption Key (DEK) Authentication**

```typescript
const authSigner: AuthSigner = {
  authenticationMethod: OdisUtils.Query.AuthenticationMethod.ENCRYPTION_KEY,
  rawKey: process.env.DEK_PRIVATE_KEY
};
```

#### Step 3: Configure ODIS Service Context

```typescript
import { OdisContextName } from "@celo/identity/lib/odis/query";

const serviceContext = OdisUtils.Query.getServiceContext(
  OdisContextName.ALFAJORES // or OdisContextName.MAINNET
);

console.log("ODIS Endpoint:", serviceContext.odisUrl);
console.log("ODIS Public Key:", serviceContext.odisPubKey);
```

#### ODIS Quota Management

**Check Current Quota**

```typescript
const { remainingQuota } = await OdisUtils.Quota.getPnpQuotaStatus(
  issuerAddress,
  authSigner,
  serviceContext
);

console.log("Remaining ODIS Quota:", remainingQuota);
```

**Purchase Quota**

```typescript
import { getContract } from "viem";
import { stableTokenABI, odisPaymentsABI } from "@celo/abis";

// Contract addresses (Alfajores testnet)
const STABLE_TOKEN_ADDRESS = "0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1"; // cUSD
const ODIS_PAYMENTS_ADDRESS = "0x645170cdB6B5c1bc80847bb728dBa56C50a20a49";

// Amount to pay (0.01 cUSD = 100 queries)
const ONE_CENT_CUSD = parseEther("0.01");

// Approve ODIS Payments to spend cUSD
const stableToken = getContract({
  address: STABLE_TOKEN_ADDRESS,
  abi: stableTokenABI,
  client: walletClient
});

const approveHash = await stableToken.write.approve([
  ODIS_PAYMENTS_ADDRESS,
  ONE_CENT_CUSD
]);

await walletClient.waitForTransactionReceipt({ hash: approveHash });

// Pay for quota
const odisPayments = getContract({
  address: ODIS_PAYMENTS_ADDRESS,
  abi: odisPaymentsABI,
  client: walletClient
});

const paymentHash = await odisPayments.write.payInCUSD([
  issuerAddress,
  ONE_CENT_CUSD
]);

await walletClient.waitForTransactionReceipt({ hash: paymentHash });

console.log("ODIS quota purchased successfully");
```

#### Verification Responsibilities

As an issuer, you must verify user ownership of identifiers. Implementation depends on identifier type:

**Phone Number Verification**

```typescript
// Example using Twilio
import twilio from "twilio";

const client = twilio(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN);

async function verifyPhoneNumber(phoneNumber: string): Promise<boolean> {
  // Send verification code
  await client.verify.v2
    .services(TWILIO_VERIFY_SERVICE_SID)
    .verifications.create({ to: phoneNumber, channel: "sms" });
  
  // User enters code (from your UI)
  const verificationCode = await getUserInput();
  
  // Check verification
  const verification = await client.verify.v2
    .services(TWILIO_VERIFY_SERVICE_SID)
    .verificationChecks.create({ to: phoneNumber, code: verificationCode });
  
  return verification.status === "approved";
}
```

**Twitter Verification**

```typescript
// Example using Twitter OAuth
async function verifyTwitterHandle(handle: string, userAddress: string): Promise<boolean> {
  // Implement OAuth flow
  const oauth = await initiateTwitterOAuth(userAddress);
  
  // User authenticates with Twitter
  const twitterUser = await completeOAuthFlow(oauth);
  
  // Verify handle matches
  return twitterUser.username === handle;
}
```

**Email Verification**

```typescript
// Example verification flow
async function verifyEmail(email: string, userAddress: string): Promise<boolean> {
  // Generate verification token
  const token = generateSecureToken();
  
  // Store token with expiry
  await storeVerificationToken(email, userAddress, token);
  
  // Send verification email
  await sendEmail(email, {
    subject: "Verify your email",
    body: `Click here to verify: ${BASE_URL}/verify/${token}`
  });
  
  // User clicks link, returns true if token is valid
  return await checkTokenVerified(token);
}
```

### SDK Integration with Viem

#### Complete Registration Example

```typescript
import { 
  createWalletClient, 
  createPublicClient, 
  http,
  parseEther,
  type Address,
  type Hex 
} from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { celoAlfajores } from "viem/chains";
import { OdisUtils } from "@celo/identity";
import { OdisContextName } from "@celo/identity/lib/odis/query";
import type { AuthSigner } from "@celo/identity/lib/odis/query";
import { getContract } from "viem";
import { federatedAttestationsABI, odisPaymentsABI, stableTokenABI } from "@celo/abis";

// Configuration
const ISSUER_PRIVATE_KEY = process.env.ISSUER_PRIVATE_KEY as Hex;
const FEDERATED_ATTESTATIONS_ADDRESS = "0x70F9314aF173c246669cFb0EEe79F9Cfd9C34ee3" as Address;
const ODIS_PAYMENTS_ADDRESS = "0x645170cdB6B5c1bc80847bb728dBa56C50a20a49" as Address;
const STABLE_TOKEN_ADDRESS = "0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1" as Address;

// Setup
const account = privateKeyToAccount(ISSUER_PRIVATE_KEY);
const walletClient = createWalletClient({
  account,
  transport: http(),
  chain: celoAlfajores
});

const publicClient = createPublicClient({
  transport: http(),
  chain: celoAlfajores
});

const issuerAddress = account.address;

// User information (provided by user after verification)
const userPlaintextIdentifier = "+12345678910";
const userAccountAddress = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb" as Address;
const attestationVerifiedTime = BigInt(Math.floor(Date.now() / 1000));

async function registerAttestation() {
  // 1. Setup authentication
  const authSigner: AuthSigner = {
    authenticationMethod: OdisUtils.Query.AuthenticationMethod.WALLET_KEY,
    sign191: ({ message, account }) => 
      walletClient.signMessage({ message, account })
  };

  const serviceContext = OdisUtils.Query.getServiceContext(
    OdisContextName.ALFAJORES
  );

  // 2. Check and top up ODIS quota if needed
  const { remainingQuota } = await OdisUtils.Quota.getPnpQuotaStatus(
    issuerAddress,
    authSigner,
    serviceContext
  );

  console.log("Remaining quota:", remainingQuota);

  if (remainingQuota < 1) {
    console.log("Purchasing ODIS quota...");
    
    // Get contract instances
    const stableToken = getContract({
      address: STABLE_TOKEN_ADDRESS,
      abi: stableTokenABI,
      client: { public: publicClient, wallet: walletClient }
    });
    
    const odisPayments = getContract({
      address: ODIS_PAYMENTS_ADDRESS,
      abi: odisPaymentsABI,
      client: { public: publicClient, wallet: walletClient }
    });
    
    const ONE_CENT_CUSD = parseEther("0.01");
    
    // Approve ODIS Payments to spend cUSD
    const approveHash = await stableToken.write.approve([
      ODIS_PAYMENTS_ADDRESS,
      ONE_CENT_CUSD
    ]);
    await publicClient.waitForTransactionReceipt({ hash: approveHash });
    
    // Pay for quota
    const paymentHash = await odisPayments.write.payInCUSD([
      issuerAddress,
      ONE_CENT_CUSD
    ]);
    await publicClient.waitForTransactionReceipt({ hash: paymentHash });
    
    console.log("ODIS quota purchased successfully");
  }

  // 3. Get obfuscated identifier from ODIS
  console.log("Getting obfuscated identifier...");
  const { obfuscatedIdentifier } = await OdisUtils.Identifier.getObfuscatedIdentifier(
    userPlaintextIdentifier,
    OdisUtils.Identifier.IdentifierPrefix.PHONE_NUMBER,
    issuerAddress,
    authSigner,
    serviceContext
  );

  console.log("Obfuscated Identifier:", obfuscatedIdentifier);

  // 4. Register attestation on-chain
  console.log("Registering attestation...");
  const federatedAttestations = getContract({
    address: FEDERATED_ATTESTATIONS_ADDRESS,
    abi: federatedAttestationsABI,
    client: { public: publicClient, wallet: walletClient }
  });

  const hash = await federatedAttestations.write.registerAttestationAsIssuer([
    obfuscatedIdentifier as Hex,
    userAccountAddress,
    attestationVerifiedTime
  ]);

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  console.log("Attestation registered!");
  console.log("Transaction:", receipt.transactionHash);
  
  return {
    obfuscatedIdentifier,
    transactionHash: receipt.transactionHash
  };
}

// Execute
registerAttestation().catch(console.error);
```

#### Complete Lookup Example

```typescript
import { createPublicClient, http, type Address, type Hex } from "viem";
import { celoAlfajores } from "viem/chains";
import { OdisUtils } from "@celo/identity";
import { OdisContextName } from "@celo/identity/lib/odis/query";
import { getContract } from "viem";
import { federatedAttestationsABI } from "@celo/abis";

const FEDERATED_ATTESTATIONS_ADDRESS = "0x70F9314aF173c246669cFb0EEe79F9Cfd9C34ee3" as Address;

const publicClient = createPublicClient({
  transport: http(),
  chain: celoAlfajores
});

async function lookupIdentifier(
  plaintextIdentifier: string,
  identifierType: string,
  trustedIssuers: Address[]
): Promise<Address[]> {
  // 1. Setup authentication for lookup
  // For read-only operations, use a zero address
  const lookupAddress = "0x0000000000000000000000000000000000000000" as Address;
  
  const authSigner = {
    authenticationMethod: OdisUtils.Query.AuthenticationMethod.WALLET_KEY,
    sign191: async () => "0x" as Hex
  };

  const serviceContext = OdisUtils.Query.getServiceContext(
    OdisContextName.ALFAJORES
  );

  // 2. Get obfuscated identifier
  const { obfuscatedIdentifier } = await OdisUtils.Identifier.getObfuscatedIdentifier(
    plaintextIdentifier,
    identifierType,
    lookupAddress,
    authSigner,
    serviceContext
  );

  console.log("Looking up:", obfuscatedIdentifier);

  // 3. Query FederatedAttestations
  const federatedAttestations = getContract({
    address: FEDERATED_ATTESTATIONS_ADDRESS,
    abi: federatedAttestationsABI,
    client: publicClient
  });

  const attestations = await federatedAttestations.read.lookupAttestations([
    obfuscatedIdentifier as Hex,
    trustedIssuers
  ]);

  const [countsPerIssuer, accounts, signers, issuedOns, publishedOns] = attestations;

  // 4. Process results
  console.log("Found attestations:");
  
  let accountIndex = 0;
  for (let i = 0; i < trustedIssuers.length; i++) {
    const count = Number(countsPerIssuer[i]);
    console.log(`\nIssuer: ${trustedIssuers[i]}`);
    console.log(`Attestation count: ${count}`);
    
    for (let j = 0; j < count; j++) {
      console.log(`  Account: ${accounts[accountIndex]}`);
      console.log(`  Signer: ${signers[accountIndex]}`);
      console.log(`  Issued: ${new Date(Number(issuedOns[accountIndex]) * 1000).toISOString()}`);
      console.log(`  Published: ${new Date(Number(publishedOns[accountIndex]) * 1000).toISOString()}`);
      accountIndex++;
    }
  }

  return accounts as Address[];
}

// Example usage
const trustedIssuers: Address[] = [
  "0x6549aF2688e07907C1b821cA44d6d65872737f05", // Kaala
  "0x388612590F8cC6577F19c9b61811475Aa432CB44"  // Libera
];

lookupIdentifier(
  "+12345678910",
  OdisUtils.Identifier.IdentifierPrefix.PHONE_NUMBER,
  trustedIssuers
).catch(console.error);
```

#### Viem Best Practices

**Use Type-Safe Contract Interactions**

```typescript
import { getContract, type Address } from "viem";
import { federatedAttestationsABI } from "@celo/abis";

// Type-safe contract instance
const contract = getContract({
  address: FEDERATED_ATTESTATIONS_ADDRESS,
  abi: federatedAttestationsABI,
  client: { public: publicClient, wallet: walletClient }
});

// TypeScript knows the exact function signatures
const hash = await contract.write.registerAttestationAsIssuer([
  obfuscatedIdentifier as `0x${string}`,
  userAddress as `0x${string}`,
  timestamp
]);
```

**Handle Hex Types Properly**

Viem uses strict `Hex` types for type safety:

```typescript
import type { Hex, Address } from "viem";

// Correct
const privateKey: Hex = process.env.PRIVATE_KEY as Hex;
const address: Address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb" as Address;

// Type assertion for obfuscated identifiers
const obfuscatedIdentifier: Hex = result.obfuscatedIdentifier as Hex;
```

**Use Proper Error Handling**

```typescript
import { 
  ContractFunctionExecutionError,
  TransactionExecutionError 
} from "viem";

try {
  const hash = await contract.write.registerAttestationAsIssuer([...]);
} catch (error) {
  if (error instanceof ContractFunctionExecutionError) {
    console.error("Contract error:", error.shortMessage);
  } else if (error instanceof TransactionExecutionError) {
    console.error("Transaction failed:", error.message);
  } else {
    console.error("Unknown error:", error);
  }
}
```

**Optimize with Public Actions**

For read-only operations, use public client directly:

```typescript
import { createPublicClient, http } from "viem";
import { celo } from "viem/chains";

const publicClient = createPublicClient({
  chain: celo,
  transport: http()
});

// No wallet needed for reads
const attestations = await publicClient.readContract({
  address: FEDERATED_ATTESTATIONS_ADDRESS,
  abi: federatedAttestationsABI,
  functionName: "lookupAttestations",
  args: [obfuscatedIdentifier, trustedIssuers]
});
```

#### Web/Browser

Browser environments require careful handling of wallet connections:

```typescript
// Check for wallet
if (window.ethereum) {
  const accounts = await window.ethereum.request({
    method: "eth_requestAccounts"
  });
  
  // Create client with injected provider
  const walletClient = createWalletClient({
    account: accounts[0],
    transport: custom(window.ethereum),
    chain: celoAlfajores
  });
}
```

**Next.js Example:**

```typescript
// pages/api/register-attestation.ts
import type { NextApiRequest, NextApiResponse } from "next";
import { OdisUtils } from "@celo/identity";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { phoneNumber, userAddress } = req.body;

  try {
    // Verify phone number (your verification logic)
    const isVerified = await verifyPhoneNumber(phoneNumber);
    
    if (!isVerified) {
      return res.status(400).json({ error: "Verification failed" });
    }

    // Register attestation
    const result = await registerAttestation(phoneNumber, userAddress);
    
    res.status(200).json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
}
```

#### Custom Identifier Types

Create custom identifier types for your use case:

```typescript
// Define custom prefix
const CUSTOM_PREFIX = "custom-app";

async function registerCustomIdentifier(
  customId: string,
  userAddress: string
) {
  const { obfuscatedIdentifier } = await OdisUtils.Identifier.getObfuscatedIdentifier(
    customId,
    CUSTOM_PREFIX, // Custom prefix
    issuerAddress,
    authSigner,
    serviceContext
  );

  // Register as usual
  await registerAttestation(obfuscatedIdentifier, userAddress);
}
```

**Best Practices for Custom Identifiers:**

* Use descriptive prefixes (e.g., `myapp://` not `ma://`)
* Document your prefix for ecosystem adoption
* Consider standardization if widely applicable
* Ensure identifiers are unique and verifiable

### Reference & Resources

#### Contract Addresses

**Mainnet (Celo)**

| Contract              | Address                                      |
| --------------------- | -------------------------------------------- |
| FederatedAttestations | `0x0aD5b1d0C25ecF6266Dd951403723B2687d6aff2` |
| OdisPayments          | `0x645170cdB6B5c1bc80847bb728dBa56C50a20a49` |
| StableToken (cUSD)    | `0x765DE816845861e75A25fCA122bb6898B8B1282a` |

**Alfajores Testnet**

| Contract              | Address                                      |
| --------------------- | -------------------------------------------- |
| FederatedAttestations | `0x70F9314aF173c246669cFb0EEe79F9Cfd9C34ee3` |
| OdisPayments          | `0x645170cdB6B5c1bc80847bb728dBa56C50a20a49` |
| StableToken (cUSD)    | `0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1` |

#### Example Repositories

| Example                    | Description                                        | Link                                                                                             |
| -------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Phone Number (Next.js)** | Web app demonstrating phone verification with Viem | [emisianto](https://github.com/celo-org/emisianto)                                               |
| **Template**               | Next.js template for Self Connect                  | [socialconnect-template](https://github.com/celo-org/socialconnect-template)                     |
| **React Native**           | Mobile app demo                                    | [SelfConnect-ReactNative-Demo](https://github.com/celo-org/SocialConnect-ReactNative-Demo)       |
| **Twitter (Client-side)**  | Twitter handle verification                        | [SelfConnect-Twitter](https://github.com/celo-org/SocialConnect-Twitter)                         |
| **Twitter (Server-side)**  | Server-side Twitter verification                   | [SelfConnect-Twitter-Server-Side](https://github.com/celo-org/SocialConnect-Twitter-Server-Side) |
| **MiniPay Nexus**          | MiniPay integration template                       | [nexus](https://github.com/celo-org/nexus)                                                       |

#### Troubleshooting

**ODIS Quota Issues**

**Problem:** "Insufficient quota" error

**Solution:**

```typescript
// Check quota
const { remainingQuota } = await OdisUtils.Quota.getPnpQuotaStatus(
  issuerAddress,
  authSigner,
  serviceContext
);

console.log("Remaining:", remainingQuota);

// Purchase more if needed (10 cUSD = 10,000 queries)
```

**Rate Limiting**

**Problem:** Too many requests to ODIS

**Solution:**

* Implement request queuing
* Cache obfuscated identifiers
* Use batch operations when possible
* Monitor quota usage

**Transaction Failures**

**Problem:** "Transaction reverted" or gas estimation failed

**Solution:**

```typescript
// Ensure sufficient gas
const hash = await contract.write.registerAttestationAsIssuer(
  [identifier, address, timestamp],
  {
    gas: 200000n // Explicit gas limit
  }
);

// Check if attestation already exists
const existing = await contract.read.lookupAttestations([
  identifier,
  [issuerAddress]
]);

if (existing.accounts.length > 0) {
  console.log("Attestation already exists");
}
```

**Network Issues**

**Problem:** RPC connection failures

**Solution:**

```typescript
// Use fallback RPCs
const transport = fallback([
  http("https://forno.celo.org"),
  http("https://rpc.ankr.com/celo"),
  http("https://1rpc.io/celo")
]);

const client = createPublicClient({
  chain: celo,
  transport
});
```

#### FAQ

**Q: Do I need to pay gas for lookups?**

A: No, lookups are read-only operations that don't require gas. You only need ODIS quota.

**Q: Can users register themselves?**

A: Users can submit the registration transaction if the issuer provides a signed attestation, but the issuer must still verify ownership and provide the obfuscated identifier.

**Q: How much does it cost to register a user?**

A: With 10 cUSD of ODIS quota, you can register 10,000 users. Gas costs for on-chain registration are typically <0.01 cUSD per transaction.

**Q: Can I trust attestations from any issuer?**

A: No, you should only trust issuers whose verification standards you trust. Each issuer is responsible for their own verification quality.

**Q: What happens if an issuer goes offline?**

A: Existing attestations remain on-chain and accessible. Users can register with other issuers for redundancy.

**Q: Can I verify multiple identifier types for the same user?**

A: Yes, you can register multiple attestations (phone + Twitter + email) for the same address, each with its own prefix.

**Q: Is E.164 format required for phone numbers?**

A: Yes, the SDK's `getObfuscatedIdentifier` function only accepts E.164 formatted phone numbers (e.g., +12345678901).

**Q: How do I map attestation results to issuers?**

A: The `lookupAttestations` return arrays are ordered by the `trustedIssuers` input array. Use `countsPerIssuer` to determine how many attestations belong to each issuer.

**Q: Can I use Self Connect on other EVM chains?**

A: Self Connect is currently designed for Celo. The contracts and ODIS infrastructure are Celo-specific.

**Q: Why use Viem instead of Web3.js or Ethers.js?**

A: Viem offers better TypeScript support, smaller bundle size, modular architecture, and is actively maintained. It's the recommended library for modern Web3 development on Celo.

**Q: Do I need to use different RPC endpoints for mainnet vs testnet?**

A: Yes, Viem's chain configurations handle this automatically:

* Mainnet: Use `celo` chain from `viem/chains`
* Testnet: Use `celoAlfajores` chain from `viem/chains`
