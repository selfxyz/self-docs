---
description: MCP servers and AI skills to help AI assistants integrate Self Pass
---

# AI Developer Tools

Self provides MCP servers and AI coding skills that give AI assistants deep context about the Self Protocol. These tools work with AI coding assistants (Claude Code, Cursor, Windsurf, Codex) as well as autonomous agent frameworks (OpenClaw, Hermes, Eliza) that support the Model Context Protocol.

## Self Protocol MCP Server

The [`@selfxyz/self-mcp`](https://github.com/selfxyz/self-mcp) server gives AI assistants comprehensive knowledge of the Self Protocol ecosystem — SDK integration, smart contracts, ZK circuits, deployed addresses, and supported documents.

### What it does

When connected, your AI assistant can:

* Integrate the Self SDK into React Native, web, and Kotlin apps
* Build custom on-chain verifier contracts using `SelfVerificationRoot`
* Set up server-side proof verification with `@selfxyz/core`
* Query on-chain registry state (merkle roots, OFAC roots)
* Understand the contract architecture and supported documents

### Setup

Add to your MCP configuration:

{% tabs %}
{% tab title="Claude Code" %}
Add to `~/.claude.json` (global) or `.mcp.json` (per-project):

```json
{
  "mcpServers": {
    "self-protocol": {
      "command": "npx",
      "args": ["@selfxyz/self-mcp"]
    }
  }
}
```
{% endtab %}

{% tab title="Cursor / Windsurf" %}
Add to your MCP configuration file:

```json
{
  "self-protocol": {
    "command": "npx",
    "args": ["@selfxyz/self-mcp"]
  }
}
```
{% endtab %}
{% endtabs %}

### Available Resources

| Resource | Description |
|----------|-------------|
| `self://overview` | Protocol architecture, verification flow, key components |
| `self://contracts` | Deployed contract addresses (mainnet + testnet) |
| `self://contracts/verifier-guide` | Building custom verifiers with `SelfVerificationRoot` |
| `self://sdk/core` | `@selfxyz/core` — server-side proof verification API |
| `self://sdk/react-native` | `@selfxyz/rn-sdk` — React Native integration |
| `self://sdk/common` | `@selfxyz/common` — shared utilities, types, `SelfAppBuilder` |
| `self://documents` | Supported document types, countries, disclosure attributes |
| `self://circuits` | ZK circuit types, signature algorithms, proof structure |

### Available Tools

| Tool | Description |
|------|-------------|
| `self_get_contract_addresses` | Get deployed contract addresses for mainnet or testnet |
| `self_check_verification` | Check if a merkle root is valid in an identity registry |
| `self_get_registry_info` | Query registry state (merkle root, OFAC roots) |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SELF_NETWORK` | `mainnet` | Network: `mainnet` or `testnet` |
| `SELF_RPC_URL` | (per network) | Custom Celo RPC URL |

## Self Protocol Skill (Claude Code)

The [`self-skill`](https://github.com/selfxyz/self-skill) is a Claude Code skill that gives Claude deep integration knowledge about Self Protocol. It includes quick start templates, integration patterns, critical gotchas, and deployed contract addresses.

### Setup

Clone the skill into your Claude Code skills directory:

```bash
git clone https://github.com/selfxyz/self-skill.git ~/.claude/skills/self-skill
```

The skill activates automatically when you mention Self Protocol, `SelfAppBuilder`, `SelfBackendVerifier`, `SelfVerificationRoot`, or related concepts.

### What it provides

* Complete frontend and backend code templates for Next.js integration
* Integration pattern guidance (off-chain, on-chain, deep linking)
* Critical gotchas (config matching, lowercase addresses, mock passport rules)
* Deployed contract addresses for Celo Mainnet and Sepolia
* Reference files for frontend SDK, backend SDK, and smart contract patterns

## Self Agent ID MCP Server

For AI agent identity (registration, verification, signing), see the separate [Self Agent ID MCP server](../agent-id/guides/mcp-user.md) documentation.
