---
description: Use Self Agent ID through MCP-compatible AI coding agents
---

# Using the MCP Server

The Self Agent ID [MCP server](https://github.com/selfxyz/self-agent-id-mcp) gives AI coding agents access to the on-chain identity registry through the [Model Context Protocol](https://modelcontextprotocol.io/). It works with Claude Code, Cursor, Windsurf, Codex, and any MCP-compatible client.

## What is MCP?

The Model Context Protocol (MCP) is an open standard that lets AI assistants access external tools and data. The Self Agent ID MCP server exposes 10 tools for agent registration, verification, authentication, and discovery.

## Configuration

Add the server to your MCP client config:

{% tabs %}
{% tab title="Claude Code" %}
Add to `~/.claude.json` (global) or `.mcp.json` (per-project):

```json
{
  "mcpServers": {
    "self-agent-id": {
      "command": "npx",
      "args": ["@selfxyz/mcp-server"],
      "env": {
        "SELF_NETWORK": "mainnet"
      }
    }
  }
}
```
{% endtab %}

{% tab title="Cursor / Windsurf" %}
Add to your MCP configuration file:

```json
{
  "mcpServers": {
    "self-agent-id": {
      "command": "npx",
      "args": ["@selfxyz/mcp-server"],
      "env": {
        "SELF_NETWORK": "mainnet",
        "SELF_AGENT_PRIVATE_KEY": "0x..."
      }
    }
  }
}
```
{% endtab %}
{% endtabs %}

### Environment Variables

| Variable | Required | Default | Description |
|----------|:--------:|---------|-------------|
| `SELF_AGENT_PRIVATE_KEY` | No | — | Agent private key (hex). Enables identity and auth tools. |
| `SELF_NETWORK` | No | `testnet` | `mainnet` or `testnet` |
| `SELF_RPC_URL` | No | Network default | Custom RPC endpoint |
| `SELF_API_URL` | No | `https://selfagentid.xyz` | Custom API base URL |

## Query-Only Mode

Without `SELF_AGENT_PRIVATE_KEY`, these tools work for looking up and verifying agents:

| Tool | Description |
|------|-------------|
| `self_lookup_agent` | Look up an agent by on-chain ID |
| `self_verify_agent` | Verify an agent's on-chain identity |
| `self_verify_request` | Verify incoming request headers |
| `self_list_agents_for_human` | List agents for a human address |
| `self_register_agent` | Start a registration flow |
| `self_check_registration` | Poll registration status |

### Example prompts

```
Look up Self Agent ID #5 on testnet.
```

```
Verify if address 0x83fa4380903fecb801F4e123835664973001ff00 is a registered agent on mainnet.
```

```
List all agents registered to 0xYourAddress on testnet.
```

## Full Mode

With `SELF_AGENT_PRIVATE_KEY` set, additional tools become available:

| Tool | Description |
|------|-------------|
| `self_get_identity` | Get the current agent's on-chain identity |
| `self_deregister_agent` | Revoke the agent's identity |
| `self_sign_request` | Generate auth headers for an HTTP request |
| `self_authenticated_fetch` | Make a signed HTTP request |

### Example prompts

```
Register me as a Self Agent on mainnet. I'll scan my passport to prove I'm human.
```

```
Sign an HTTP POST request to https://api.example.com/data with body {"query": "test"}.
```

```
Make an authenticated request to https://api.example.com/protected.
```

## Resources

The server exposes two MCP resources:

| URI | Description |
|-----|-------------|
| `self://networks` | Contract addresses, chain IDs, and RPC URLs |
| `self://identity` | Current agent's on-chain identity (requires key) |

## Integration Prompt

The server includes a built-in prompt for generating verification middleware:

```
Use the self_integrate_verification prompt with Express to generate verification middleware.
```

This generates framework-specific code using `SelfAgentVerifier` for Express, FastAPI, Flask, or Axum.

## Example Workflows

### Register and verify

1. "Register me as a Self Agent on testnet"
2. Scan passport when prompted
3. "Check my registration status"
4. "What's my agent identity?"

### Verify a peer

1. "Is agent 0x83fa... registered on mainnet?"
2. "What credentials does Agent ID #5 have?"

### Build a gated API

1. "Use the self_integrate_verification prompt to generate Express middleware"
2. Customize the generated code for your routes

## Next Steps

- [MCP Server Repository](https://github.com/selfxyz/self-agent-id-mcp)
- [Build an agent](agent-builder.md)
- [Verify agents in your API](service-operator.md)
- [Troubleshooting](../troubleshooting.md)
