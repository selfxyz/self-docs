# Table of contents

* [Welcome to Self](README.md)

## Get started

* [What is Self Enterprise](self-enterprise/get-started/what-is-self-enterprise.md)
* [Quickstart](self-enterprise/get-started/quickstart.md)
* [Core concepts](self-enterprise/get-started/concepts.md)
* [How verification works](self-enterprise/get-started/how-it-works.md)

## Dashboard

* [Overview](self-enterprise/dashboard/overview.md)
* [Configure a product](self-enterprise/dashboard/configure-a-product.md)
* [Activity log](self-enterprise/dashboard/activity-log.md)
* [API keys](self-enterprise/dashboard/api-keys.md)
* [Webhooks](self-enterprise/dashboard/webhooks.md)
* [People (members & invites)](self-enterprise/dashboard/people.md)
* [Billing](self-enterprise/dashboard/billing.md)

## SDK

* [Node.js / TypeScript](self-enterprise/sdk/nodejs.md)
* [Verify webhooks](self-enterprise/sdk/verify-webhooks.md)
* [Error handling](self-enterprise/sdk/error-handling.md)
* [API reference](self-enterprise/sdk/api-reference.md)

## Webhooks

* [Overview](self-enterprise/webhooks/overview.md)
* [Signature verification](self-enterprise/webhooks/signature-verification.md)
* [Event catalog](self-enterprise/webhooks/events.md)
* [Best practices](self-enterprise/webhooks/best-practices.md)

## Verification flows

* [Anatomy of a flow](self-enterprise/flows/anatomy.md)
* [Disclosures](self-enterprise/flows/disclosures.md)
* [Supported documents](self-enterprise/flows/supported-documents.md)
* [Test vs. live](self-enterprise/flows/test-vs-live.md)
* [Using mock passports](self-enterprise/guides/using-mock-passports.md)

## Billing

* [Plans](self-enterprise/billing/plans.md)
* [Credits and usage](self-enterprise/billing/credits-and-usage.md)

## Migrate

* [From the open-source SDK](self-enterprise/migration/from-self-pass-sdk.md)

## Reference

* [Document specifications](self-enterprise/reference/document-specifications/README.md)
  * [Aadhaar](self-enterprise/reference/document-specifications/aadhaar.md)
  * [KYC](self-enterprise/reference/document-specifications/kyc.md)
* [Supported countries](self-enterprise/reference/supported-countries.md)
* [Troubleshooting](self-enterprise/reference/troubleshooting.md)

## Self Connect

* [Introduction & Overview](self-connect/introduction-and-overview.md)
* [Architecture & How It Works](self-connect/architecture-and-how-it-works.md)
* [Developer Guide](self-connect/developer-guide.md)

## Self Agent ID

* [Overview](agent-id/overview.md)
* [Register Without the Web App](agent-id/register-without-the-app.md)
* [Registration Modes](agent-id/registration-modes.md)
* [SDK Integration](agent-id/sdk-integration.md)
* [Verification Patterns](agent-id/verification-patterns.md)
* [Smart Contracts](agent-id/smart-contracts.md)
* [ERC-8004 & Proof-of-Human](agent-id/erc-8004.md)
* [Agent Registration JSON & Agent Cards](agent-id/agent-registration-json.md)
* [REST API](agent-id/rest-api.md)
* [CLI](agent-id/cli.md)
* [Celo Agent Visa](agent-id/celo-agent-visa.md)
* [Guides](agent-id/guides/agent-builder.md)
  * [Building an Agent](agent-id/guides/agent-builder.md)
  * [Verifying Agents (Service)](agent-id/guides/service-operator.md)
  * [Gating Smart Contracts](agent-id/guides/contract-developer.md)
  * [Using MCP Server](agent-id/guides/mcp-user.md)
* [Troubleshooting](agent-id/troubleshooting.md)

## Self Pass · legacy

* [Overview](self-pass/README.md)
* [Quickstart](self-pass/quickstart.md)
* [Disclosures](self-pass/disclosures.md)
* [Use Deeplinking](self-pass/use-deeplinking.md)
* [Using Mock Passports](self-pass/using-mock-passports.md)
* [Frontend SDK](self-pass/frontend/qrcode-sdk.md)
  * [API Reference](self-pass/frontend/qrcode-sdk-api-reference.md)
  * [Disclosure Configs](self-pass/frontend/disclosure-configs.md)
* [Backend SDK](self-pass/backend/basic-integration.md)
  * [ConfigStore](self-pass/backend/configstore.md)
  * [API Reference](self-pass/backend/selfbackendverifier-api-reference.md)
* [Smart Contracts](self-pass/contracts/basic-integration.md)
  * [Deployed Contracts](self-pass/contracts/deployed-contracts.md)
  * [Airdrop Example](self-pass/contracts/airdrop-example.md)
  * [Happy Birthday Example](self-pass/contracts/happy-birthday-example.md)
  * [Working with userDefinedData](self-pass/contracts/working-with-userdefineddata.md)
* [Mobile SDK](self-pass/mobile-sdk/getting-started.md)
  * [SelfClient Provider Setup](self-pass/mobile-sdk/selfclient-provider.md)
  * [Native Modules Setup](self-pass/mobile-sdk/native-modules-setup.md)
  * [Onboarding Screen Components](self-pass/mobile-sdk/onboarding-screens.md)
  * [Examples](self-pass/mobile-sdk/examples/README.md)
    * [Minimal Setup](self-pass/mobile-sdk/examples/minimal-setup.md)
    * [Navigation Setup](self-pass/mobile-sdk/examples/navigation-setup.md)
    * [Demo Walkthrough](self-pass/mobile-sdk/examples/demo-walkthrough.md)
* [KMP SDK (Alpha)](self-pass/kmp-sdk.md)
* [Document Specifications](self-pass/document-specification/aadhaar.md)
  * [Aadhaar](self-pass/document-specification/aadhaar.md)
  * [KYC](self-pass/document-specification/kyc.md)
* [Supported Countries](self-pass/architecture/countries-list.md)
* [Troubleshooting](self-pass/troubleshooting.md)
* [AI Developer Tools](self-pass/ai-developer-tools.md)
* [V1 to V2 Migration Guide](self-pass/migration-v1-v2.md)
* [Architecture](self-pass/architecture/overview.md)
  * [ZK Proof Architecture](self-pass/architecture/zk-proof-architecture.md)
  * [Verification in the IdentityVerificationHub](self-pass/architecture/verification-hub.md)
  * [OFAC & CSCA Auto-Updaters](self-pass/architecture/ofac-csca-auto-updaters.md)
  * [Verification Result](self-pass/architecture/self-attestation.md)
  * [Deployments](self-pass/architecture/deployments.md)
