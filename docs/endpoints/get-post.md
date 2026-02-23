# Get Post

Retrieve details and publishing status of a post group.

## Endpoint

```
GET https://api.publora.com/api/v1/get-post/:postGroupId
```

## Headers

| Header | Required | Description |
|--------|----------|-------------|
| `x-publora-key` | Yes | Your API key |
| `x-publora-user-id` | No | Managed user ID (workspace only) |

## Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `postGroupId` | string | The post group ID returned from create-post |

## Response

```json
{
  "success": true,
  "postGroupId": "507f1f77bcf86cd799439011",
  "posts": [
    {
      "platform": "twitter",
      "platformId": "123456789",
      "content": "Excited to share our new product launch! 🚀",
      "status": "published",
      "postedId": "1234567890123456789"
    },
    {
      "platform": "linkedin",
      "platformId": "ABC123",
      "content": "Excited to share our new product launch! 🚀",
      "status": "published",
      "postedId": "urn:li:share:7654321"
    }
  ]
}
```

### Failed Post Response

When a post fails, the response includes detailed error information:

```json
{
  "success": true,
  "postGroupId": "507f1f77bcf86cd799439011",
  "posts": [
    {
      "platform": "threads",
      "platformId": "17841412345678",
      "content": "Check out our new feature!",
      "status": "failed",
      "error": {
        "code": "PLATFORM_AUTH_EXPIRED",
        "message": "Error validating access token",
        "platformStatusCode": 401,
        "platformError": "Invalid OAuth access token",
        "failedAt": "2026-02-22T14:30:00.000Z",
        "retryable": false
      }
    }
  ]
}
```

## Post Status Values

| Status | Meaning |
|--------|---------|
| `draft` | Saved but not yet scheduled |
| `scheduled` | Waiting for scheduled time |
| `pending` | Being processed by scheduler |
| `processing` | Currently publishing to platform |
| `published` | Successfully posted |
| `failed` | Publishing failed (see `error` object for details) |

## Error Codes

When `status` is `failed`, the `error` object contains:

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Error classification code (see table below) |
| `message` | string | Human-readable error message |
| `platformStatusCode` | number/null | HTTP status code from the platform API |
| `platformError` | string/null | Raw error message from the platform |
| `failedAt` | string | Timestamp when the failure occurred |
| `retryable` | boolean | Whether the error might succeed on retry |

| Error Code | Description | Retryable |
|------------|-------------|-----------|
| `PLATFORM_AUTH_EXPIRED` | Access token expired or revoked | No |
| `RATE_LIMITED` | Platform rate limit exceeded | Yes |
| `INVALID_CONTENT` | Content rejected by platform (e.g., too long, banned words) | No |
| `CONTENT_TOO_LARGE` | Media file exceeds platform limits | No |
| `PLATFORM_SERVER_ERROR` | Platform API returned 5xx error | Yes |
| `NETWORK_ERROR` | Could not reach platform API | Yes |
| `TIMEOUT_ERROR` | Request timed out | Yes |
| `UNKNOWN_ERROR` | Unclassified error | Maybe |

## Examples

### JavaScript (fetch)

```javascript
const postGroupId = '507f1f77bcf86cd799439011';
const response = await fetch(
  `https://api.publora.com/api/v1/get-post/${postGroupId}`,
  { headers: { 'x-publora-key': 'YOUR_API_KEY' } }
);
const data = await response.json();

for (const post of data.posts) {
  console.log(`${post.platform}: ${post.status}`);
  if (post.postedId) {
    console.log(`  Platform post ID: ${post.postedId}`);
  }
}
```

### Python (requests)

```python
import requests

post_group_id = '507f1f77bcf86cd799439011'
response = requests.get(
    f'https://api.publora.com/api/v1/get-post/{post_group_id}',
    headers={'x-publora-key': 'YOUR_API_KEY'}
)
data = response.json()

for post in data['posts']:
    print(f"{post['platform']}: {post['status']}")
```

### cURL

```bash
curl https://api.publora.com/api/v1/get-post/507f1f77bcf86cd799439011 \
  -H "x-publora-key: YOUR_API_KEY"
```

### Node.js (axios)

```javascript
const axios = require('axios');

const { data } = await axios.get(
  `https://api.publora.com/api/v1/get-post/${postGroupId}`,
  { headers: { 'x-publora-key': 'YOUR_API_KEY' } }
);
// Check if all posts published successfully
const allPublished = data.posts.every(p => p.status === 'published');
console.log(`All published: ${allPublished}`);
```

## Errors

| Status | Error | Cause |
|--------|-------|-------|
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 404 | `"Post group not found"` | Invalid ID or post belongs to another user |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
