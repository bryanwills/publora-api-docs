# Telegram API - Post to Channel via REST API

Post to Telegram channels and groups programmatically using the Publora REST API. A simpler alternative to the Telegram Bot API, Telethon, Pyrogram, or MTProto libraries.

## Telegram API Overview

Publora provides a unified REST API for publishing content to Telegram channels and groups via bot or MTProto connection. Supports rich text formatting with markdown syntax, images, and videos. No need to manage Telegram bot tokens directly, handle MTProto complexity, or implement message queuing. Publora supports two connection types: **bot** (using a BotFather token) and **mtproto** (using a user session), each with different capabilities and limits.

### Why Use Publora Instead of Telegram Bot API / Telethon / Pyrogram?

| Feature | Publora API | Telegram Bot API / Telethon |
|---------|-------------|----------------------------|
| Authentication | Single API key | Bot token + channel setup |
| Message formatting | Markdown support | Markdown/HTML support |
| Media handling | Automatic | Manual upload |
| Multi-platform | Post to 11 platforms | Telegram only |
| Setup time | 5 minutes | 15-30 minutes |
| Rate limiting | Handled | Manual implementation |

### Keywords: Telegram API, Telegram Bot API, Telegram channel API, Telegram posting API, post to Telegram programmatically, Telegram REST API, Telegram developer API, Telegram automation API, Telegram message API, send message Telegram API, Telethon alternative, Pyrogram alternative, MTProto API

## Platform ID Format

```
telegram-{chatId}
```

Where `{chatId}` is your Telegram channel or group chat ID assigned during bot connection setup.

## Requirements

- A Telegram bot created via [@BotFather](https://t.me/BotFather)
- The bot's token and target channel/group name provided during setup
- The bot must be added as an **administrator** of the target channel or group with the **`can_post_messages`** permission explicitly enabled (being an admin alone is not enough)
- API key from Publora

## Connection Types

Publora supports two connection types for Telegram:

### Bot Connection (Default)

1. Create a bot with [@BotFather](https://t.me/BotFather) on Telegram
2. Copy the bot token provided by BotFather
3. Add the bot as an administrator to your target channel or group
4. In the Publora dashboard, provide the bot token and channel name (e.g., `@mychannel`)
5. Publora will verify the connection and assign a platform ID

### MTProto Connection

MTProto connections use a Telegram **user session** instead of a bot, enabling higher limits (4,096-character captions, 4 GB file uploads). MTProto requires a **Telegram Premium** subscription on the service user account.

**Service user concept:** The MTProto connection operates through a dedicated Telegram user account (the "service user") that acts on behalf of your organization. This user must be an admin of the target channel/group. Because the service user is a regular Telegram account (not a bot), it benefits from the higher user-tier limits. The service user's session is stored encrypted by Publora in the connection record and decrypted at runtime.

> **Note:** MTProto connections require the service user to have an active Telegram Premium subscription. Without Premium, certain features (such as the extended 4,096-character caption limit) will not be available. For self-hosted deployments, the `TELEGRAM_MT_ALLOW_NON_PREMIUM` environment variable can be set to bypass the Premium requirement for MTProto connections.

## Supported Content

| Type | Supported | Limits |
|------|-----------|--------|
| Text | Yes | 4,096 characters (text-only messages, both bot and MTProto); 1,024 characters (captions with media, bot API) |
| Images | Yes | JPEG, PNG, GIF, BMP, WebP (WebP auto-converted to JPEG) |
| Videos | Yes | MP4, MOV formats |
| Formatting | Yes | Bold, italic, code, links, blockquotes |

## Text Formatting

Telegram supports markdown-style formatting in message text. You can use the following syntax in your `content`:

| Syntax | Result | Example |
|--------|--------|---------|
| `*text*` | **Bold** | `*important*` |
| `_text_` | _Italic_ | `_emphasis_` |
| `` `code` `` | `Inline code` | `` `variable` `` |
| ```` ```code``` ```` | Code block | ```` ```print("hello")``` ```` |
| `[text](url)` | Hyperlink | `[click here](https://example.com)` |
| `> text` | Blockquote | `> quoted text` |

## Caption vs. Message

When posting with media (images or videos), the text content is sent as a **caption** attached to the media. Captions have a stricter limit of 1,024 characters when using the bot API (MTProto supports 4,096-character captions). Text-only messages (without media) support up to 4,096 characters for both bot API and MTProto connections.

## Examples

### Post a Text Message

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: '*Product Update v2.5*\n\nWe have shipped the following improvements:\n\n- _Faster API response times_ (avg 45ms)\n- New `batch` endpoint for bulk operations\n- Improved error messages with `error_code` field\n\n> This update is backward compatible. No migration needed.\n\nFull changelog: [docs.example.com/changelog](https://docs.example.com/changelog)',
    platforms: ['telegram-1001234567890']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

content = """*Product Update v2.5*

We have shipped the following improvements:

- _Faster API response times_ (avg 45ms)
- New `batch` endpoint for bulk operations
- Improved error messages with `error_code` field

> This update is backward compatible. No migration needed.

Full changelog: [docs.example.com/changelog](https://docs.example.com/changelog)"""

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': content,
        'platforms': ['telegram-1001234567890']
    }
)

data = response.json()
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "*Product Update v2.5*\n\nWe have shipped the following improvements:\n\n- _Faster API response times_ (avg 45ms)\n- New `batch` endpoint for bulk operations\n- Improved error messages with `error_code` field\n\n> This update is backward compatible. No migration needed.\n\nFull changelog: [docs.example.com/changelog](https://docs.example.com/changelog)",
    "platforms": ["telegram-1001234567890"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const content = `*Product Update v2.5*

We have shipped the following improvements:

- _Faster API response times_ (avg 45ms)
- New \`batch\` endpoint for bulk operations
- Improved error messages with \`error_code\` field

> This update is backward compatible. No migration needed.

Full changelog: [docs.example.com/changelog](https://docs.example.com/changelog)`;

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content,
  platforms: ['telegram-1001234567890']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

### Post with an Image

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: '*New Dashboard Preview*\n\nHere is a sneak peek at our redesigned analytics dashboard. Key improvements include real-time data refresh and customizable widgets.',
    platforms: ['telegram-1001234567890']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': '*New Dashboard Preview*\n\nHere is a sneak peek at our redesigned analytics dashboard. Key improvements include real-time data refresh and customizable widgets.',
        'platforms': ['telegram-1001234567890']
    }
)

data = response.json()
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "*New Dashboard Preview*\n\nHere is a sneak peek at our redesigned analytics dashboard. Key improvements include real-time data refresh and customizable widgets.",
    "platforms": ["telegram-1001234567890"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: '*New Dashboard Preview*\n\nHere is a sneak peek at our redesigned analytics dashboard. Key improvements include real-time data refresh and customizable widgets.',
  platforms: ['telegram-1001234567890']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

> **Note:** To attach media to a Telegram post, first create the post, then upload media using the [media upload workflow](../guides/media-uploads.md) with the returned `postGroupId`.

### Post with a Video

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: '*Feature Demo*\n\nWatch our 60-second demo of the new collaboration tools.',
    platforms: ['telegram-1001234567890']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': '*Feature Demo*\n\nWatch our 60-second demo of the new collaboration tools.',
        'platforms': ['telegram-1001234567890']
    }
)

data = response.json()
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "*Feature Demo*\n\nWatch our 60-second demo of the new collaboration tools.",
    "platforms": ["telegram-1001234567890"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: '*Feature Demo*\n\nWatch our 60-second demo of the new collaboration tools.',
  platforms: ['telegram-1001234567890']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

## Platform Quirks

- **Bot must be admin**: The Telegram bot must be added as an administrator to the target channel or group with `can_post_messages` permission. The admin permission check happens at **connection time**, not at posting time — if the bot is not an admin with `can_post_messages`, the connection will be rejected during setup.
- **Caption overflow behavior**: When posting with media, the text is sent as a caption (1,024-character limit via bot API). If the content exceeds 1,024 characters and media is attached, the text is sent as a separate reply message to the media post rather than being truncated or rejected. Text-only messages support up to 4,096 characters via MTProto.
- **WebP auto-conversion**: WebP images are automatically converted to JPEG before sending to Telegram.
- **Markdown formatting**: Telegram supports a subset of markdown. Publora's `safeParseMarkdown` uses single asterisks for bold: `*bold*`, `_italic_`, `` `code` ``, `[text](url)`, and `> blockquote`. Note that this differs from standard Markdown where `**double asterisks**` denote bold. Not all markdown features are supported (e.g., headers are not rendered as headers).
- **Mixed media types not supported**: A single post cannot contain both images and videos. Attempting to send mixed media types will result in an error. Note that Publora's backend validation does **not** enforce mixed media type restrictions for Telegram (it only does so for Instagram). If Telegram rejects mixed media, the error comes from the Telegram API itself, not from Publora's validation layer.
- **Bot connection media group limitations**: Bot connections support either a single video OR multiple photos in a media group, but not multiple videos. Multi-video posts are not supported for bot connections. If you need to post multiple videos, create separate posts for each video.
- **Channel name format**: When setting up the connection, provide the channel name with the `@` prefix (e.g., `@mychannel`). The `channelName` field also accepts a numeric chat ID, which is required for private channels that do not have a public `@username`.
- **Bot token security**: Your bot token is stored by Publora in the connection record and is never exposed in API responses. Treat your bot token like a password. Note: encryption at rest for bot tokens is planned but not yet implemented. MTProto sessions, however, ARE encrypted at rest (the service user's session is stored encrypted and decrypted at runtime).
- **MTProto test-connection limitation**: The `test-connection` endpoint always fails for MTProto connections. The validator checks `connectionType === 'bot'` before validating the bot token. MTProto connections have `connectionType: 'mtproto'`, so they skip the bot validation block entirely and fall through to the default failure response with a misleading `'Invalid bot token'` error. This is a known limitation; MTProto connections work correctly for posting despite failing the test-connection check.
- **Silent messages**: Telegram supports silent message delivery, but this feature is not currently available through the Publora API.
- **Message editing**: Once posted, Telegram messages can be edited, but this capability is not currently exposed through the Publora API.

## API Limits

Understanding the difference between Bot API limits and regular user limits is critical for successful Telegram integration.

### Character Limits

| Element | Limit |
|---------|-------|
| Text message | 4,096 characters |
| **Bot caption** | **1,024 characters** (bots cannot get Premium 4,096 limit) |
| Bot username | 32 characters |
| Channel name | 128 characters |

### Image Limits (Bot API)

| Property | Limit |
|----------|-------|
| Max size | 10 MB |
| Max count | 10 images per media group |
| Formats | JPEG, PNG, GIF, WebP, BMP |

### Video Limits (Bot API)

| Property | Limit |
|----------|-------|
| Duration | No limit (only size matters) |
| **Max size** | **50 MB** (NOT 4 GB - that's for regular users!) |
| Local Bot API Server | 2 GB max |
| Formats | MP4, MOV |

### CRITICAL: Bot API vs User Limits

Bots have significantly lower limits than regular Telegram users. This is a common source of errors:

| Limit | Bot API | Regular Users |
|-------|---------|---------------|
| File size | 50 MB | 4 GB |
| Caption | 1,024 chars | 4,096 chars |

### Common Error Messages

| Error | Cause |
|-------|-------|
| `MEDIA_CAPTION_TOO_LONG` | Caption exceeds 1,024 characters (bots) |
| `Bad Request: file is too big` | File exceeds 50 MB |

### Rate Limits

| Scope | Limit |
|-------|-------|
| Global | 30 messages/second |
| Per group | 20 messages/minute |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
