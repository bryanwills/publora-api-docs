# Post Multiple Threads Posts via API - Threads Threading Guide

Post multi-part threads (connected posts) to Meta's Threads platform programmatically using the Publora REST API. Create threaded conversations without managing the Threads API complexity.

> **⚠️ Temporary Restriction:** Multi-threaded nested posts are temporarily unavailable due to Threads app reconnection status. This guide documents the feature for when it returns. Single posts, carousel posts, and standalone threads continue to work. Contact support@publora.com for updates.

## Threads Multi-Post API Overview

On Threads (by Meta), you can create connected posts that appear as a conversation thread. This requires:
1. Creating a media container for the first post
2. Publishing it
3. Creating subsequent posts with `reply_to_id` pointing to the previous post
4. Publishing each in sequence

Publora handles this entire flow with a single API call.

### Keywords: Threads API, Threads multi-post API, Meta Threads API, post to Threads API, Threads thread API, Threads reply chain API, Threads automation API, create Threads post programmatically, Threads bot API, Instagram Threads API, publish multiple Threads posts

## Quick Example

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: `I spent 3 months rebuilding our entire backend. Here's what I learned about technical debt.

When we started, our codebase had grown organically over 4 years. What began as a simple MVP had become a tangled mess of shortcuts, deprecated patterns, and "temporary" fixes that became permanent.

The first step was painful: mapping every dependency. We discovered circular imports, dead code paths, and services that nobody remembered building. This audit alone took 3 weeks.

The rebuild itself taught us that incremental migration beats big-bang rewrites. We ran old and new systems in parallel, comparing outputs in real-time. When something broke, we could instantly rollback.

Key lesson: Technical debt isn't just slow code. It's the cognitive load on every engineer who touches your codebase. The rebuild cut our onboarding time from 2 weeks to 3 days.`,
    platforms: ['threads-789']
  })
});
```

Publora will automatically split this into multiple connected Threads posts.

## How Threads Threading Works

### The Official Threads API Way

Using Meta's Graph API directly requires multiple steps:

```javascript
// Step 1: Create first post container
const container1 = await fetch(
  `https://graph.threads.net/me/threads?media_type=TEXT&text=First+post&access_token=${token}`,
  { method: 'POST' }
);
const { id: containerId1 } = await container1.json();

// Step 2: Publish first post
const post1 = await fetch(
  `https://graph.threads.net/me/threads_publish?creation_id=${containerId1}&access_token=${token}`,
  { method: 'POST' }
);
const { id: postId1 } = await post1.json();

// Step 3: Create second post with reply_to_id
const container2 = await fetch(
  `https://graph.threads.net/me/threads?media_type=TEXT&text=Second+post&reply_to_id=${postId1}&access_token=${token}`,
  { method: 'POST' }
);
// ... and so on
```

### The Publora Way

```javascript
// Single request - everything handled automatically
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: "First post\n\n---\n\nSecond post\n\n---\n\nThird post",
    platforms: ['threads-789']
  })
});
```

## Thread Formatting Options

### Automatic Splitting

Content over 500 characters is automatically split:
- At paragraph breaks (`\n\n`) first
- At sentence endings (`. `, `! `, `? `) second
- At word boundaries as fallback
- Unlike Twitter, no `[1/N]` markers are added by default

### Manual Thread Parts with `---`

```javascript
const content = `Hot take: Most "productivity" advice is just procrastination in disguise.

---

We spend hours optimizing our Notion setup instead of doing the actual work. We read about how successful people structure their days instead of structuring our own.

---

The most productive people I know have embarrassingly simple systems. A paper notebook. A basic to-do list. Maybe a calendar.

---

The tool doesn't matter. The work does. Stop optimizing, start shipping.`;
```

### Manual Thread Parts with Markers

You can include markers if you want them in your posts:

```javascript
const content = `Hot take: Most "productivity" advice is procrastination in disguise. [1/4]

We spend hours optimizing systems instead of doing actual work. [2/4]

The most productive people have embarrassingly simple systems. [3/4]

The tool doesn't matter. The work does. Start shipping. [4/4]`;
```

## Character Limits

| Element | Limit |
|---------|-------|
| Post body | 500 characters |
| Hashtags | 1 per post (platform limit) |
| Thread parts | No fixed limit |

## Adding Media to Threads

Media attaches to the **first post** only:

```javascript
// Step 1: Create thread
const postResponse = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: `Just launched our new product! Here's a quick walkthrough.

---

The main feature is the dashboard. It shows all your metrics in real-time with zero configuration.

---

We're offering 50% off for early adopters. Link in bio!`,
    platforms: ['threads-789']
  })
});

const { postGroupId } = await postResponse.json();

// Step 2: Upload image carousel (goes to first post)
const formData = new FormData();
formData.append('file', screenshot1);
formData.append('postGroupId', postGroupId);

await fetch('https://api.publora.com/api/v1/upload-media', {
  method: 'POST',
  headers: { 'x-publora-key': 'YOUR_API_KEY' },
  body: formData
});
```

### Media Options

| Type | Limit | Notes |
|------|-------|-------|
| Images | Up to 10 (carousel) | WebP auto-converted to JPEG |
| Video | 1 | MP4 format required |
| Carousel | Up to 10 items | Images only |

## Scheduling Threads Posts

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: `Your long thread content...`,
    platforms: ['threads-789'],
    scheduledTime: '2026-03-15T09:00:00.000Z'
  })
});
```

## Rate Limits

The Threads API has publishing limits:

| Limit Type | Value |
|------------|-------|
| Posts per 24 hours | 250 |
| Posts per hour | ~25 (approximate) |

**Important:** Each post in a thread counts toward these limits. A 5-post thread uses 5 posts from your daily quota.

## Error Handling

### Partial Thread Failure

If some posts succeed but others fail:

```json
{
  "status": "partially_published",
  "error": "Thread partially published: 3/5 posts. Error: Rate limit exceeded",
  "publishedIds": ["123", "456", "789"],
  "headPostId": "123"
}
```

Published posts remain live. Publora tracks which parts succeeded.

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Rate limit | Too many posts | Wait and retry |
| Media processing | Video still encoding | Retry after delay |
| Invalid media | Unsupported format | Use JPEG/PNG/MP4 |

## Python Example

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': '''Unpopular opinion: You don't need a personal brand.

---

What you need is to be genuinely helpful and share what you're learning. That's it.

---

The "build your personal brand" industry has convinced us that we need logos, color palettes, and "content pillars."

---

But the creators I actually follow? They just share interesting stuff consistently. No strategy deck required.

---

Be useful. Be consistent. Be yourself. That's the whole playbook.''',
        'platforms': ['threads-789']
    }
)

print(response.json())
```

## cURL Example

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "First post of my thread\n\n---\n\nSecond post continues the thought\n\n---\n\nFinal post wraps it up!",
    "platforms": ["threads-789"]
  }'
```

## Why Use Publora for Threads?

| Feature | Publora | Threads API Direct |
|---------|---------|-------------------|
| Thread creation | Single API call | Multiple sequential calls |
| OAuth handling | Not required | Complex Meta OAuth flow |
| Content splitting | Automatic | Manual implementation |
| API access | Instant | Requires Meta app review |
| Multi-platform | Yes (+ Twitter, LinkedIn, etc.) | Threads only |

## Platform Quirks

- **Single hashtag limit:** Threads allows maximum 1 hashtag per post
- **WebP auto-conversion:** Publora converts WebP images automatically
- **No edit support:** Posted content cannot be edited via API
- **Video processing:** Videos may take time to process before publishing

## Related

- [Threading Guide](./threading.md) - Complete threading documentation
- [Threads Platform Reference](../platforms/threads.md) - Full platform details
- [Scheduling Posts](./scheduling.md) - Schedule threads for later
- [Media Uploads](./media-uploads.md) - Add images and carousels

---

*Post multi-part threads to Meta's Threads with a single API call using [Publora](https://publora.com).*
