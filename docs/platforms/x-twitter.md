# Twitter API / X API - Post Tweets via REST API

Post to Twitter (X) programmatically using the Publora REST API. A simpler alternative to the official Twitter API v2, Tweepy, node-twitter-api-v2, or Twitter Java SDK.

## Twitter API Overview

Publora provides a unified REST API for posting tweets to X (formerly Twitter), including text posts, media attachments, images, videos, and automatic thread splitting for long-form content. No need to manage OAuth tokens, API rate limits, or complex Twitter API authentication flows.

### Why Use Publora Instead of Twitter API v2 / Tweepy / node-twitter-api-v2?

| Feature | Publora API | Twitter API v2 / Tweepy |
|---------|-------------|-------------------------|
| Authentication | Single API key | Complex OAuth 2.0 flow |
| Rate limit handling | Automatic | Manual implementation |
| Thread creation | Automatic splitting | Manual tweet chaining |
| Multi-platform | Post to 10 platforms | Twitter only |
| Setup time | 5 minutes | Hours to days |
| Pricing | Free tier + $2.99/account | Free tier + paid tiers |

### Keywords: Twitter API, X API, post tweet API, Twitter posting API, tweet programmatically, Twitter bot API, Twitter automation API, send tweet API, Twitter REST API, Twitter developer API

## Platform ID Format

```
twitter-{userId}
```

Where `{userId}` is your X/Twitter numeric user ID assigned during account connection.

## Requirements

- An X/Twitter account connected via OAuth through the Publora dashboard
- API key from Publora

## Supported Content

| Type | Supported | Limits |
|------|-----------|--------|
| Text | Yes | 280 characters (25,000 for Premium) |
| Images | Yes | Up to 4 per post, PNG preferred |
| Videos | Yes | MP4 format |
| Threads | Yes | Auto-split with `[1/N]` markers |

## Character Counting

X has specific rules for character counting that Publora handles automatically:

- Standard characters count as 1
- Emojis count as **2 characters**
- URLs are shortened to 23 characters regardless of length
- Publora calculates the true character count before posting

## Threading

When your content exceeds the character limit, Publora automatically splits it into a thread (multiple connected tweets):

### How It Works

Publora uses the official X API v2 `reply.in_reply_to_tweet_id` parameter to chain tweets together. Each subsequent tweet is posted as a reply to the previous one, creating a connected thread.

**Technical flow:**
1. First tweet is published normally via `POST /2/tweets`
2. Each subsequent tweet is published with `reply.in_reply_to_tweet_id` set to the previous tweet's ID
3. All tweets share the same `conversation_id` (equals the first tweet's ID)

### Automatic Splitting

When content exceeds the character limit (280 for standard, 25,000 for Premium), Publora automatically:

- Splits at paragraph breaks (`\n\n`) when possible
- Falls back to sentence boundaries (`. `, `! `, `? `)
- Falls back to word boundaries if needed
- Adds `[1/N]` markers at the end of each tweet (e.g., `[1/3]`, `[2/3]`, `[3/3]`)
- Reserves ~8 characters per tweet for the marker

### Manual Thread Parts

You can manually define where thread breaks should occur using either method:

**Method 1: Triple dash separator**
```
This is my first tweet in the thread.

---

This is my second tweet in the thread.

---

And this is my third tweet!
```

**Method 2: Explicit markers**
```
First part of the thread [1/3]

Second part of the thread [2/3]

Third and final part [3/3]
```

When explicit `[n/m]` markers are detected, Publora preserves them exactly as written and splits at those points.

### Media in Threads

- **Images:** Up to 4 images attached to the first tweet only
- **Video:** Single video attached to the first tweet only
- Subsequent tweets in the thread are text-only
- Images and video cannot be combined in the same tweet

## Examples

### Post a Text Update

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Hello from Publora! Posting to X has never been easier.',
    platforms: ['twitter-12345678']
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
        'content': 'Hello from Publora! Posting to X has never been easier.',
        'platforms': ['twitter-12345678']
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
    "content": "Hello from Publora! Posting to X has never been easier.",
    "platforms": ["twitter-12345678"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Hello from Publora! Posting to X has never been easier.',
  platforms: ['twitter-12345678']
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
    content: 'Check out this screenshot of our new dashboard!',
    platforms: ['twitter-12345678']
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
        'content': 'Check out this screenshot of our new dashboard!',
        'platforms': ['twitter-12345678']
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
    "content": "Check out this screenshot of our new dashboard!",
    "platforms": ["twitter-12345678"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Check out this screenshot of our new dashboard!',
  platforms: ['twitter-12345678']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

> **Note:** To attach media to a post, first create the post, then use the [media upload workflow](../guides/media-uploads.md) with the returned `postGroupId`.

### Post a Thread

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: `Here is a deep dive into our new feature release and what it means for developers building on our platform.

We have completely redesigned the API layer to support batch operations, real-time webhooks, and granular rate limiting. This means you can now process up to 1000 requests per minute with predictable throughput.

The new SDK is available for JavaScript, Python, and Go. Each SDK includes full TypeScript definitions, async support, and built-in retry logic. Check our docs for migration guides.`,
    platforms: ['twitter-12345678']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

content = """Here is a deep dive into our new feature release and what it means for developers building on our platform.

We have completely redesigned the API layer to support batch operations, real-time webhooks, and granular rate limiting. This means you can now process up to 1000 requests per minute with predictable throughput.

The new SDK is available for JavaScript, Python, and Go. Each SDK includes full TypeScript definitions, async support, and built-in retry logic. Check our docs for migration guides."""

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': content,
        'platforms': ['twitter-12345678']
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
    "content": "Here is a deep dive into our new feature release and what it means for developers building on our platform.\n\nWe have completely redesigned the API layer to support batch operations, real-time webhooks, and granular rate limiting. This means you can now process up to 1000 requests per minute with predictable throughput.\n\nThe new SDK is available for JavaScript, Python, and Go. Each SDK includes full TypeScript definitions, async support, and built-in retry logic. Check our docs for migration guides.",
    "platforms": ["twitter-12345678"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const content = `Here is a deep dive into our new feature release and what it means for developers building on our platform.

We have completely redesigned the API layer to support batch operations, real-time webhooks, and granular rate limiting. This means you can now process up to 1000 requests per minute with predictable throughput.

The new SDK is available for JavaScript, Python, and Go. Each SDK includes full TypeScript definitions, async support, and built-in retry logic. Check our docs for migration guides.`;

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content,
  platforms: ['twitter-12345678']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

Publora will automatically split this into a numbered thread (e.g., `[1/3]`, `[2/3]`, `[3/3]`) at sentence boundaries.

## Platform Quirks

- **Emoji character counting**: Each emoji counts as 2 characters toward the 280-character limit. Publora accounts for this automatically.
- **PNG preferred for images**: While JPEG works, PNG images tend to render with higher quality on X due to their compression algorithm.
- **Thread numbering**: Publora adds `[1/N]` markers at the end of each tweet in a thread. This is appended after the content, so it reduces available character space by approximately 6-8 characters per tweet.
- **Premium character limit**: If your account has X Premium, the character limit increases to 25,000. Publora detects this based on your connected account.
- **Rate limits**: X enforces its own rate limits. If you hit them, Publora will return the appropriate error from the X API.
- **Up to 4 images**: You can attach a maximum of 4 images to a single tweet. Attempting to attach more will result in an error.
- **Video and images are mutually exclusive**: A single tweet can contain either images or a video, but not both.

## API Limits

### Character Limit

- **Standard accounts:** 280 characters
- **Premium accounts:** 25,000 characters
- **Threading:** Supported - content over 280 characters will be split into a thread

### Image Limits

| Property | Limit |
|----------|-------|
| Max size | 5 MB |
| Max count | 4 per tweet |
| Formats | JPEG, PNG, GIF, WebP |

### Video Limits (API)

| Property | Limit |
|----------|-------|
| **Max duration** | **2 minutes (120 seconds)** |
| Max size | 512 MB |
| Formats | MP4, MOV |

> **Important:** The X API has a **2-minute video limit** even though the native X app allows videos up to 2:20 (140 seconds). If you attempt to upload a longer video via the API, you will receive the error: *"This user is not allowed to post a video longer than 2 minutes"*

**Premium users via native app:** Up to 4 hours (not available via API)

## Character Limits

| Account Type | Limit |
|-------------|-------|
| Standard | 280 characters |
| X Premium | 25,000 characters |
| Thread tweet (each part) | Same as above, minus `[X/N]` marker space (~8 chars) |

## Rate Limits

The underlying X API v2 has the following publishing limits based on your account tier:

| Tier | Monthly Posts | Per 15 Minutes | Per 24 Hours |
|------|--------------|----------------|--------------|
| Free | 500 | ~17 | ~500 |
| Basic ($100/mo) | 10,000 | 100 per user | 10,000 per app |
| Pro ($5,000/mo) | 1,000,000 | Higher | Higher |

Publora returns the appropriate error from the X API if rate limits are exceeded. Each tweet in a thread counts as a separate post toward these limits.


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
