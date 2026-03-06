# Using Publora with Context7

Access up-to-date Publora API documentation directly in your AI coding assistant using Context7.

## What is Context7?

Context7 is an MCP (Model Context Protocol) server by Upstash that provides real-time documentation to AI coding assistants. Instead of AI models relying on potentially outdated training data, Context7 fetches current documentation on demand.

**Key benefits:**
- **Always up-to-date** — Get current Publora API docs, not stale training data
- **Code examples included** — Real, working code snippets from official docs
- **Works everywhere** — Cursor, Claude Code, Windsurf, and any MCP-compatible tool

## Quick Start

Add "use context7" to your prompt to get Publora documentation:

```text
How do I schedule a post to Twitter with Publora? use context7
```

```text
Show me how to upload media with Publora API. use context7
```

```text
What are Publora's rate limits? use context7
```

## Supported AI Tools

### Claude Code (CLI)

Context7 is available as a built-in MCP server in Claude Code:

```text
claude> Schedule a LinkedIn post with an image using Publora. use context7
```

### Cursor

Configure Context7 in your Cursor MCP settings:

```json
// .cursor/mcp.json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

Then use in your prompts:

```text
@context7 How do I create a thread on Twitter with Publora?
```

### Windsurf

Add to your Windsurf MCP configuration:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

### Other MCP-Compatible Tools

Any tool supporting the Model Context Protocol can use Context7. Check your tool's documentation for MCP server configuration.

## Example Prompts

### Getting Started

```text
Show me a complete example of creating a scheduled post with Publora. use context7
```

### Platform-Specific

```text
How do I post a Reel to Instagram with Publora API? use context7
```

```text
What are the TikTok-specific settings in Publora? use context7
```

### Media Uploads

```text
Step-by-step guide to uploading images to Publora for scheduled posts. use context7
```

### Threading

```text
How do I create a Twitter thread using Publora API? use context7
```

### Analytics

```text
How do I get LinkedIn post statistics with Publora? use context7
```

### Error Handling

```text
What error codes does Publora return and how should I handle them? use context7
```

## How It Works

1. **You ask a question** with "use context7" in your prompt
2. **Context7 identifies** that you're asking about Publora
3. **Fetches current docs** from Publora's indexed documentation
4. **AI generates response** using the up-to-date information

This means:
- New API features are immediately available in AI responses
- Deprecated patterns are avoided
- Code examples match current API versions

## Publora Documentation Coverage

Context7 indexes all Publora documentation including:

| Category | Topics |
|----------|--------|
| Getting Started | Authentication, API keys, first requests |
| Endpoints | create-post, update-post, delete-post, get-post, upload-media, platform-connections |
| Platforms | Twitter/X, LinkedIn, Instagram, Threads, TikTok, YouTube, Facebook, Bluesky, Mastodon, Telegram |
| Guides | Scheduling, threading, media uploads, cross-platform posting, error handling, rate limits |
| Examples | JavaScript, Python, TypeScript, Go, PHP, Ruby, cURL |

## Tips for Better Results

### 1. Be Specific

```text
# Good - specific question
How do I add images to a LinkedIn carousel post with Publora? use context7

# Less effective - vague question
How do I use Publora? use context7
```

### 2. Mention Your Stack

```text
Create a Next.js API route to schedule posts with Publora. use context7
```

```text
Python script to bulk import posts from CSV using Publora. use context7
```

### 3. Ask for Complete Examples

```text
Show me a complete example with error handling for creating a scheduled post with Publora. use context7
```

### 4. Request Step-by-Step

```text
Step-by-step guide to uploading a video and scheduling a TikTok post with Publora. use context7
```

## Comparison: With vs Without Context7

### Without Context7

The AI might:
- Use outdated API endpoints
- Reference deprecated parameters
- Generate code that doesn't match current API
- Miss platform-specific settings

### With Context7

The AI will:
- Use current API endpoints and parameters
- Follow latest authentication patterns
- Include accurate rate limit information
- Reference correct platform-specific options

## Troubleshooting

### Context7 Not Finding Publora Docs

Try being more specific:

```text
# Instead of
"How do I post?"

# Use
"How do I create a post with Publora API? use context7"
```

### Getting Generic Responses

Make sure to include "use context7" at the end of your prompt:

```text
Create a function to schedule a Twitter post. use context7
```

### MCP Server Not Working

Verify your configuration:

1. Check MCP config file exists (`.cursor/mcp.json` for Cursor)
2. Ensure npx can run without issues
3. Restart your AI tool after config changes

## Related Guides

- [Using Publora with Cursor AI](./cursor-ai.md)
- [Getting Started](../getting-started.md)
- [API Authentication](../authentication.md)

---

*Access up-to-date [Publora](https://publora.com) documentation in any AI coding assistant with Context7.*
