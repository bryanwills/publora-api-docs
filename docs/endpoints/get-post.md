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
| `x-publora-client` | No | Client identifier (e.g., `"mcp"`) |

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
      "_id": "663a1b2c3d4e5f6a7b8c9d01",
      "platform": "twitter",
      "platformId": "123456789",
      "content": "Excited to share our new product launch! 🚀",
      "status": "published",
      "postedId": "1234567890123456789"
    },
    {
      "_id": "663a1b2c3d4e5f6a7b8c9d02",
      "platform": "linkedin",
      "platformId": "ABC123",
      "content": "Excited to share our new product launch! 🚀",
      "status": "published",
      "postedId": "urn:li:share:7654321"
    }
  ]
}
```

> **Note:** With `.lean()`, the `error` field may be **absent** (undefined) for non-failed posts if the error subdocument was never populated — it will not necessarily be `null`. When a post has failed, the `error` field contains an error object with details about the failure.

> **Note:** For draft and scheduled posts, the `postedId` field will be absent from the response since the post has not yet been published to the platform. Only published posts include a `postedId`.

> **Note:** The `platformId` field returns the raw platform ID (e.g., `"123456789"`), not the compound format used by list-posts (e.g., `"twitter-123456789"`). See the [list-posts](./list-posts) endpoint for details on this difference.

### Failed Post Response

When a post fails, the response includes detailed error information:

```json
{
  "success": true,
  "postGroupId": "507f1f77bcf86cd799439011",
  "posts": [
    {
      "_id": "663a1b2c3d4e5f6a7b8c9d03",
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

> **Note:** The statuses above apply to **individual platform posts**. The post **group** itself can also have a status of `partially_published`, which occurs when some platform posts in the group succeeded while others failed. For example, if a post group targets Twitter, LinkedIn, and Threads, and the Threads post fails while the others succeed, the group status will be `partially_published`. Query individual posts within the group to determine which platforms succeeded or failed.

## Error Codes

When `status` is `failed`, the `error` object contains:

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Error classification code (see table below) |
| `message` | string | Human-readable error message |
| `platformStatusCode` | number/undefined | HTTP status code from the platform API (field may be absent rather than explicitly `null`) |
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

> **Note:** The error codes listed above are not exhaustive. The schema does not validate error codes, so additional codes beyond these eight may appear in the response. Always handle unknown codes gracefully.

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
| 400 | `"Invalid x-publora-user-id"` | The `x-publora-user-id` header value is not a valid ObjectId format |
| 401 | `"API key is required"` | Missing `x-publora-key` header |
| 401 | `"Invalid API key"` | The provided API key is not valid |
| 401 | `"Invalid API key owner"` | The API key exists but its owner account could not be found |
| 403 | `"API access is not enabled for this account"` | The account does not have API access enabled |
| 403 | `"MCP access is not enabled for this account"` | The account does not have MCP access enabled (MCP-only keys) |
| 403 | `"Workspace access is not enabled for this key"` | The API key does not have workspace/managed-user permissions |
| 403 | `"User is not managed by key"` | The `x-publora-user-id` references a user not managed by this API key |
| 404 | `"Post group not found"` | Invalid ID or post belongs to another user |
| 500 | `"Failed to fetch post group"` | Malformed post group ID or internal server error |
| 500 | `"Internal server error"` | Unexpected server error in middleware |

> **Note:** If `x-publora-user-id` matches the API key owner, no workspace check is triggered — the header is effectively a no-op in that case.

> **Note:** If the `postGroupId` is not a valid MongoDB ObjectId format (e.g., too short, contains invalid characters), the server returns a **500** error (`"Failed to fetch post group"`) instead of a **400** validation error. Ensure you pass only valid ObjectId strings received from the create-post or list-posts endpoints.


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
