# Publora MCP Server

The **Publora MCP server** lets AI assistants — Claude, Cursor, OpenClaw, and any MCP-compatible client — schedule and publish social media posts across 10 platforms (X, LinkedIn, Instagram, Threads, TikTok, YouTube, Facebook, Bluesky, Mastodon, Telegram) through natural language.

This is the general MCP section. For a specific client, jump to its setup guide below.

## Endpoint & transport

- **URL:** `https://mcp.publora.com/mcp` (Streamable HTTP)
- **Remote / hosted** — nothing to run locally.
- A health check is available at `https://mcp.publora.com/health`.

## Authentication

Publora MCP uses **static API keys** (starting with `sk_`), not OAuth. Send your key on every request using either header:

```
x-publora-key: sk_your_key_here
```

or

```
Authorization: Bearer sk_your_key_here
```

You cannot create API keys programmatically — generate one in your dashboard. See [Getting Started](/getting-started).

## Set up your client

| Client | Guide |
|--------|-------|
| **Cursor** | [Cursor AI setup](/guides/cursor-ai) |
| **Claude & generic MCP clients** | [Client setup](/mcp/client-setup) |
| **OpenClaw** | [OpenClaw integration](/mcp/openclaw) |
| **Any client (overview)** | [MCP server guide](/guides/mcp-server) |

## Tools

The server exposes 11 tools — connections, post create/get/update/delete, listing, media upload, and LinkedIn reactions/comments. Full schemas: [Tools reference](/mcp/tools-reference).

## Examples & troubleshooting

- [Examples](/mcp/examples)
- [Troubleshooting](/mcp/troubleshooting)

## Discovery

Registries and clients can discover this server's metadata and tools without authenticating via the [MCP Server Card](https://mcp.publora.com/.well-known/mcp/server-card.json) (`/.well-known/mcp/server-card.json`).
