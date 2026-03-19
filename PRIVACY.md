# Privacy Policy — easy-notion-mcp

**Last updated:** 2026-03-19
**Operator:** Grey Iris

## What this integration does

easy-notion-mcp is an open-source MCP (Model Context Protocol) server that lets
AI agents interact with your Notion workspace using markdown. It acts as a bridge
between MCP-compatible clients (like Claude) and Notion's API.

## Data we access

When you authorize easy-notion-mcp, it accesses only the Notion pages and databases
you explicitly select during the OAuth consent screen. We request these capabilities:

- Read, update, and insert page/database content
- Read and insert comments
- Read user information (names and profile images, not email addresses)

## Data we store

**Self-hosted mode (default):** Your Notion API token is stored locally on your
machine in your MCP client's config file. No data is sent to any server we operate.
The integration runs entirely on your device.

**OAuth mode (when using HTTP transport):** Your OAuth access token and refresh
token are stored on the server instance you connect to. If you self-host, tokens
are stored in a local encrypted file. We do not operate a public hosted instance
at this time.

## Data we share

We do not share, sell, or transmit your Notion data to any third party. All API
calls go directly between your machine (or your self-hosted instance) and Notion's
API servers.

## Data retention

Tokens are stored only as long as the integration is active. Revoking access in
Notion (Settings → Connections) immediately invalidates the token. Uninstalling the
MCP server removes all local configuration.

## Open source

This integration is fully open source under the MIT license. You can audit the
source code at https://github.com/Grey-Iris/easy-notion-mcp.

## Contact

For privacy questions: open an issue at https://github.com/Grey-Iris/easy-notion-mcp/issues
