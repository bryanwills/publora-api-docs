# Error Handling

This guide covers how to handle errors from the Publora API, including HTTP status codes, error response formats, retry strategies, and dealing with partial failures.

## How It Works

The Publora API uses standard HTTP status codes to indicate success or failure. When an error occurs, the response body contains a JSON object with details about what went wrong.

### HTTP Status Codes

| Code | Meaning | Description |
|---|---|---|
| `200` | OK | Request succeeded (GET, PUT, DELETE, and most POST endpoints including `create-post`) |
| `201` | Created | Resource created successfully (used by specific POST endpoints such as workspace user creation) |
| `400` | Bad Request | Invalid input, missing required fields, or malformed data |
| `401` | Unauthorized | Missing or invalid API key |
| `403` | Forbidden | Valid API key but insufficient permissions or plan limits reached |
| `404` | Not Found | Resource does not exist or does not belong to your account |
| `409` | Conflict | Duplicate or conflicting operation |
| `500` | Internal Server Error | Something went wrong on Publora's side |

### Error Response Format

Errors are returned as JSON in one of two formats:

```json
{
  "error": "Human-readable error message"
}
```

or

```json
{
  "message": "Detailed description of what went wrong"
}
```

### Common Errors

| Status | Error Message | Cause | Resolution |
|---|---|---|---|
| `401` | `"API key is required"` | The `x-publora-key` header is missing | Include a valid API key in the `x-publora-key` header |
| `401` | `"Invalid API key"` | The `x-publora-key` header contains an invalid or revoked key | Verify your API key is correct |
| `401` | `"Invalid API key owner"` | Workspace resolution failed for the API key (the key exists but its owner account could not be resolved) | Verify the API key is associated with a valid account, or regenerate the key |
| `403` | `"API access is not enabled for this account"` | Your account is on the Starter plan, which does not include API access | Upgrade to Pro or Premium to use the API |
| `403` | `"Workspace access is not enabled for this key"` | The API key does not have workspace-level access enabled | Verify the API key has the correct permissions, or generate a new key with workspace access |
| `403` | `"Subscription required"` | Your account does not have an active subscription. Response includes `message: "No active plan entitlements found for this account."` | Subscribe to a paid plan |
| `403` | `"Account on hold"` | Your account has been placed on hold. Response includes `message: "Your account is temporarily on hold. Please contact support."`, `holdExpiresAt` (ISO 8601 timestamp), and `holdReason` fields | Wait until the hold expires or contact support to resolve the issue |
| `403` | `"Account inactive"` | Your account has been deactivated. Response includes `message: "Your account is inactive."` | Contact support@publora.com to reactivate your account |
| `403` | Limit exceeded (structured) | You have exceeded a usage limit for your plan. Response uses `LimitExceededError` format with fields: `code`, `error` (short label), `message` (long text), `metric`, `limit`, `used`, `requested`, `remaining`, `periodStart`, `periodEnd`, `planName`, plus context-specific fields (see below) | Wait for the current period to reset, reduce usage, or upgrade your plan |
| `403` | `"MCP access is not enabled for this account"` | Your account does not have MCP access enabled | Contact support or upgrade your plan to enable MCP access |
| `400` | `"Invalid x-publora-user-id"` | The `x-publora-user-id` header contains an invalid or non-existent user ID | Verify the user ID is correct and belongs to the account associated with the MCP API key |
| `400` | `"Invalid scheduled time format"` | The `scheduledTime` is not a valid ISO 8601 string | Use a valid time in the format `YYYY-MM-DDTHH:mm:ss.sssZ` |
| `404` | `"Post group not found"` | The post group ID does not exist or does not belong to your account | Verify the ID and ensure you are using the correct API key |
| `400` | `"Cannot update post: post is currently in {status} status"` | Attempting to modify a post that is in a non-editable status (e.g., `published`, `failed`) | Only `draft` and `scheduled` posts can be updated. The external API `update-post` endpoint only checks `postGroup.status` (must be `draft` or `scheduled`). It does NOT check `processingStatus`. The `processingStatus` check only applies to the internal dashboard update flow. Note: `processing` is not a value of the `status` field -- it is tracked separately via the `processingStatus` field (values: `pending`, `processing`, `finished`). The `status` field values are: `draft`, `scheduled`, `published`, `failed`, `partially_published`. |

### LimitExceededError Codes

The `code` field in a `LimitExceededError` response identifies the specific limit that was exceeded:

| Code | Description |
|---|---|
| `POST_LIMIT_REACHED` | Monthly post limit for the plan has been reached |
| `SCHEDULED_POST_LIMIT_REACHED` | Maximum number of pending scheduled posts reached |
| `SCHEDULE_HORIZON_REACHED` | Scheduled time exceeds the plan's maximum scheduling horizon |
| `PLATFORM_NOT_AVAILABLE` | The target platform is not available on the current plan |
| `CONNECTIONS_OVER_LIMIT` | Number of connected accounts exceeds the plan limit |
| `CHANNEL_LIMIT_REACHED` | Maximum number of channels for the plan has been reached |

### LimitExceededError Context Fields

In addition to the standard fields (`code`, `error`, `message`, `metric`, `limit`, `used`, `requested`, `remaining`, `periodStart`, `periodEnd`, `planName`), the response may include context-specific fields depending on the error code:

| Context Field | Appears With | Description |
|---|---|---|
| `overLimitBy` | `CONNECTIONS_OVER_LIMIT` | How many units over the limit the request would go |
| `disallowedPlatforms` | `PLATFORM_NOT_AVAILABLE` | Array of platforms in the request that are not available on the current plan |
| `allowedPlatforms` | `PLATFORM_NOT_AVAILABLE` | Array of platforms that are available on the current plan |
| `projectedUsed` | `CHANNEL_LIMIT_REACHED` | The projected usage count if the operation were to proceed |
| `blockedPlatforms` | `POST_LIMIT_REACHED`, `SCHEDULED_POST_LIMIT_REACHED` | Array of objects with `platformSelection`, `used`, and `remaining` fields. Present when scope is connection-level |
| `scope` | Various | The scope of the limit (e.g., `"account"`, `"connection"`) |
| `scheduledTime` | `SCHEDULE_HORIZON_REACHED` | The requested scheduled time that exceeded the horizon |
| `maxScheduledDate` | `SCHEDULE_HORIZON_REACHED` | The latest allowed scheduled date for the current plan |

### Threads-Specific Errors

| Error Message | Cause | Resolution |
|---|---|---|
| `"Multi-part threads (nested replies) are temporarily blocked while we wait for Meta to approve additional permissions for our app."` | Multi-part threads (content >500 chars or with `---` separators) are temporarily disabled due to Threads API access requirements | Keep content under 500 characters without `---` separators. Use carousel posts for multiple images. Contact support@publora.com for updates. |

**Note:** Single posts (under 500 characters) and carousel posts on Threads continue to work normally. Only multi-part threads (where content is automatically split into multiple connected replies) are temporarily restricted.

### Post-Level Errors (Partial Failures)

A post group can target multiple platforms. Even if some platforms succeed, others may fail. This results in a `partially_published` status on the post group.

When checking a post group's status, each individual platform post has its own `status` field:

```json
{
  "success": true,
  "postGroupId": "664f1a2b3c4d5e6f7a8b9c0d",
  "posts": [
    {
      "platform": "twitter",
      "platformId": "123456",
      "status": "published",
      "content": "Check out our latest product launch!",
      "postedId": "1234567890123456789"
    },
    {
      "platform": "tiktok",
      "platformId": "789012",
      "status": "failed",
      "error": {
        "code": "PLATFORM_VALIDATION_ERROR",
        "message": "Video FPS is below minimum requirement",
        "platformStatusCode": null,
        "platformError": null,
        "failedAt": "2026-03-15T14:01:12.000Z",
        "retryable": false
      }
    },
    {
      "platform": "linkedin",
      "platformId": "ABCDEF",
      "status": "published",
      "content": "Check out our latest product launch!",
      "postedId": "urn:li:share:7000000000000000000"
    }
  ]
}
```

**Note:** The `content` and `status` fields exist on each individual post within the `posts` array, not at the post group level. The top-level response only contains `success`, `postGroupId`, and `posts`.

> **`error` field format:** The `error` field on individual platform posts is a structured object, not a plain string. The `get-post` endpoint returns the full error object with fields: `code`, `message`, `platformStatusCode`, `platformError`, `failedAt`, and `retryable`.

> **`platformId` vs platform ID string:** In the `get-post` response, `platform` and `platformId` are separate fields. The `platform` field contains the platform name (e.g., `"twitter"`), and `platformId` contains just the account identifier (e.g., `"123456"`). This is different from the `{platform}-{id}` combined format (e.g., `"twitter-123456"`) used in the `platforms` array when creating posts.

### Platform-Specific Validation Errors

Some platforms have strict media requirements that are checked before publishing:

- **TikTok:** Minimum FPS requirement, specific video format requirements, duration limits
- **Instagram:** Video aspect ratio and duration requirements for Reels
- **YouTube:** Video format and size requirements

These errors appear at the individual platform post level, not at the HTTP response level.

## Examples

### Error Handling Wrapper (JavaScript)

```javascript
class PubloraApiError extends Error {
  constructor(status, message, body) {
    super(message);
    this.name = 'PubloraApiError';
    this.status = status;
    this.body = body;
  }
}

async function publoraRequest(endpoint, options = {}) {
  const url = `https://api.publora.com/api/v1${endpoint}`;

  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY',
      ...options.headers
    }
  });

  const body = await response.json();

  if (!response.ok) {
    const message = body.error || body.message || 'Unknown API error';
    throw new PubloraApiError(response.status, message, body);
  }

  return body;
}

// Usage
try {
  const post = await publoraRequest('/create-post', {
    method: 'POST',
    body: JSON.stringify({
      content: 'Hello from Publora!',
      platforms: ['twitter-123456'],
      scheduledTime: '2026-03-15T14:00:00.000Z'
    })
  });

  console.log('Post created:', post.postGroupId);
} catch (error) {
  if (error instanceof PubloraApiError) {
    switch (error.status) {
      case 401:
        console.error('Authentication failed. Check your API key.');
        break;
      case 403:
        console.error('Access denied:', error.message);
        // Could be "Subscription required", limit exceeded, or account hold
        break;
      case 400:
        console.error('Bad request:', error.message);
        // Could be "Invalid scheduled time" or validation errors
        break;
      case 404:
        console.error('Resource not found:', error.message);
        break;
      case 409:
        console.error('Conflict:', error.message);
        break;
      case 500:
        console.error('Server error. Retry later.');
        break;
      default:
        console.error(`API error (${error.status}):`, error.message);
    }
  } else {
    console.error('Network or unexpected error:', error.message);
  }
}
```

### Error Handling Wrapper (Python)

```python
import requests


class PubloraApiError(Exception):
    def __init__(self, status_code, message, body):
        super().__init__(message)
        self.status_code = status_code
        self.body = body


class PubloraClient:
    BASE_URL = 'https://api.publora.com/api/v1'

    def __init__(self, api_key):
        self.session = requests.Session()
        self.session.headers.update({
            'Content-Type': 'application/json',
            'x-publora-key': api_key
        })

    def _request(self, method, endpoint, **kwargs):
        url = f'{self.BASE_URL}{endpoint}'
        response = self.session.request(method, url, **kwargs)

        try:
            body = response.json()
        except ValueError:
            body = {}

        if not response.ok:
            message = body.get('error') or body.get('message') or 'Unknown API error'
            raise PubloraApiError(response.status_code, message, body)

        return body

    def create_post(self, content, platform_ids, scheduled_time=None):
        payload = {
            'content': content,
            'platforms': platform_ids
        }
        if scheduled_time:
            payload['scheduledTime'] = scheduled_time

        return self._request('POST', '/create-post', json=payload)

    def get_post(self, post_group_id):
        return self._request('GET', f'/get-post/{post_group_id}')


# Usage
client = PubloraClient('YOUR_API_KEY')

try:
    post = client.create_post(
        content='Hello from Publora!',
        platforms=['twitter-123456'],
        scheduled_time='2026-03-15T14:00:00.000Z'
    )
    print(f"Post created: {post['postGroupId']}")

except PubloraApiError as e:
    if e.status_code == 401:
        print('Authentication failed. Check your API key.')
    elif e.status_code == 403:
        print(f'Access denied: {e}')
    elif e.status_code == 400:
        print(f'Bad request: {e}')
    elif e.status_code == 404:
        print(f'Resource not found: {e}')
    elif e.status_code == 500:
        print('Server error. Retry later.')
    else:
        print(f'API error ({e.status_code}): {e}')

except requests.RequestException as e:
    print(f'Network error: {e}')
```

### cURL with Error Checking

```bash
#!/bin/bash

API_KEY="YOUR_API_KEY"
BASE_URL="https://api.publora.com/api/v1"

# Make a request and capture both HTTP status and body
HTTP_RESPONSE=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/create-post" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Hello from Publora!",
    "platforms": ["twitter-123456"],
    "scheduledTime": "2026-03-15T14:00:00.000Z"
  }')

# Split response body and status code
HTTP_BODY=$(echo "$HTTP_RESPONSE" | sed '$d')
HTTP_STATUS=$(echo "$HTTP_RESPONSE" | tail -1)

case $HTTP_STATUS in
  200|201)
    echo "Success:"
    echo "$HTTP_BODY" | jq .
    ;;
  400)
    echo "Bad request:"
    echo "$HTTP_BODY" | jq '.error // .message'
    ;;
  401)
    echo "Authentication failed. Check your x-publora-key header."
    ;;
  403)
    echo "Access denied:"
    echo "$HTTP_BODY" | jq '.error // .message'
    ;;
  404)
    echo "Not found:"
    echo "$HTTP_BODY" | jq '.error // .message'
    ;;
  500)
    echo "Server error. Retry later."
    ;;
  *)
    echo "Unexpected status: $HTTP_STATUS"
    echo "$HTTP_BODY"
    ;;
esac
```

### Node.js (axios) with Error Handling

```javascript
const axios = require('axios');

const api = axios.create({
  baseURL: 'https://api.publora.com/api/v1',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

// Add a response interceptor for centralized error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response) {
      const { status, data } = error.response;
      const message = data.error || data.message || 'Unknown error';

      switch (status) {
        case 401:
          console.error('Authentication failed. Check your API key.');
          break;
        case 403:
          console.error('Access denied:', message);
          break;
        case 400:
          console.error('Bad request:', message);
          break;
        case 404:
          console.error('Not found:', message);
          break;
        case 500:
          console.error('Server error. Retry later.');
          break;
        default:
          console.error(`API error (${status}):`, message);
      }
    } else if (error.request) {
      console.error('No response received. Check your network connection.');
    } else {
      console.error('Request setup error:', error.message);
    }

    return Promise.reject(error);
  }
);

// Usage
try {
  const { data: post } = await api.post('/create-post', {
    content: 'Hello from Publora!',
    platforms: ['twitter-123456'],
    scheduledTime: '2026-03-15T14:00:00.000Z'
  });

  console.log('Post created:', post.postGroupId);
} catch (error) {
  // Error already logged by the interceptor
  // Add any additional handling here
}
```

---

### Retry Logic with Exponential Backoff

Server errors (`500`) and network failures are usually transient. Implement retry logic for these cases.

**JavaScript (fetch)**

```javascript
async function publoraRequestWithRetry(endpoint, options = {}, maxRetries = 3) {
  const url = `https://api.publora.com/api/v1${endpoint}`;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          'x-publora-key': 'YOUR_API_KEY',
          ...options.headers
        }
      });

      const body = await response.json();

      if (response.ok) {
        return body;
      }

      // Do not retry client errors (4xx) -- they will not succeed on retry
      if (response.status >= 400 && response.status < 500) {
        throw new Error(`Client error ${response.status}: ${body.error || body.message}`);
      }

      // Retry on server errors (5xx)
      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
        console.log(`Server error (${response.status}). Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }

      throw new Error(`Server error ${response.status} after ${maxRetries} retries`);

    } catch (error) {
      // Retry on network errors
      if (error.name === 'TypeError' && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000;
        console.log(`Network error. Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }

      throw error;
    }
  }
}

// Usage
try {
  const post = await publoraRequestWithRetry('/create-post', {
    method: 'POST',
    body: JSON.stringify({
      content: 'Retryable post!',
      platforms: ['twitter-123456'],
      scheduledTime: '2026-03-15T14:00:00.000Z'
    })
  });

  console.log('Post created:', post.postGroupId);
} catch (error) {
  console.error('Failed after retries:', error.message);
}
```

**Python (requests)**

```python
import time
import requests


def publora_request_with_retry(method, endpoint, max_retries=3, **kwargs):
    url = f'https://api.publora.com/api/v1{endpoint}'
    headers = {
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    }

    for attempt in range(max_retries + 1):
        try:
            response = requests.request(method, url, headers=headers, **kwargs)
            body = response.json()

            if response.ok:
                return body

            # Do not retry client errors (4xx)
            if 400 <= response.status_code < 500:
                message = body.get('error') or body.get('message') or 'Unknown error'
                raise Exception(f'Client error {response.status_code}: {message}')

            # Retry on server errors (5xx)
            if attempt < max_retries:
                delay = (2 ** attempt)  # 1s, 2s, 4s
                print(f'Server error ({response.status_code}). Retrying in {delay}s...')
                time.sleep(delay)
                continue

            raise Exception(f'Server error {response.status_code} after {max_retries} retries')

        except requests.ConnectionError:
            if attempt < max_retries:
                delay = (2 ** attempt)
                print(f'Network error. Retrying in {delay}s...')
                time.sleep(delay)
                continue
            raise


# Usage
try:
    post = publora_request_with_retry(
        'POST', '/create-post',
        json={
            'content': 'Retryable post!',
            'platforms': ['twitter-123456'],
            'scheduledTime': '2026-03-15T14:00:00.000Z'
        }
    )
    print(f"Post created: {post['postGroupId']}")
except Exception as e:
    print(f'Failed after retries: {e}')
```

**Node.js (axios)**

```javascript
const axios = require('axios');

async function publoraRequestWithRetry(method, endpoint, data = null, maxRetries = 3) {
  const config = {
    method,
    url: `https://api.publora.com/api/v1${endpoint}`,
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    },
    data
  };

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await axios(config);
      return response.data;
    } catch (error) {
      const status = error.response?.status;

      // Do not retry client errors
      if (status && status >= 400 && status < 500) {
        const message = error.response.data?.error || error.response.data?.message;
        throw new Error(`Client error ${status}: ${message}`);
      }

      // Retry on server errors or network errors
      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000;
        const reason = status ? `Server error (${status})` : 'Network error';
        console.log(`${reason}. Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }

      throw error;
    }
  }
}

// Usage
try {
  const post = await publoraRequestWithRetry('POST', '/create-post', {
    content: 'Retryable post!',
    platforms: ['twitter-123456'],
    scheduledTime: '2026-03-15T14:00:00.000Z'
  });

  console.log('Post created:', post.postGroupId);
} catch (error) {
  console.error('Failed after retries:', error.message);
}
```

---

### Checking for Partial Failures

After a post group has been processed, check whether all platforms succeeded or if some failed.

**JavaScript (fetch)**

```javascript
async function checkPostGroupStatus(postGroupId) {
  const response = await fetch(
    `https://api.publora.com/api/v1/get-post/${postGroupId}`,
    {
      headers: { 'x-publora-key': 'YOUR_API_KEY' }
    }
  );

  const postGroup = await response.json();

  // Determine overall status from individual posts
  const posts = postGroup.posts || [];
  const statuses = posts.map(p => p.status);
  const allPublished = statuses.every(s => s === 'published');
  const allFailed = statuses.every(s => s === 'failed');
  const overallStatus = allPublished ? 'published' : allFailed ? 'failed' : 'partially_published';

  console.log(`Post group ${postGroup.postGroupId}: ${overallStatus}`);

  if (overallStatus === 'partially_published') {
    console.log('Some platforms failed:');

    const failed = postGroup.posts.filter(p => p.status === 'failed');
    const succeeded = postGroup.posts.filter(p => p.status === 'published');

    console.log(`  Succeeded: ${succeeded.length}`);
    for (const p of succeeded) {
      console.log(`    - ${p.platform}: ${p.postedId}`);
    }

    console.log(`  Failed: ${failed.length}`);
    for (const p of failed) {
      console.log(`    - ${p.platform}: ${p.error}`);
    }

    return { status: 'partial', succeeded, failed };
  }

  if (overallStatus === 'failed') {
    console.log('All platforms failed.');
    for (const p of posts) {
      console.log(`  - ${p.platform}: ${p.error}`);
    }
    return { status: 'failed', succeeded: [], failed: posts };
  }

  if (overallStatus === 'published') {
    console.log('All platforms succeeded.');
    return { status: 'success', succeeded: posts, failed: [] };
  }

  // Still processing or scheduled (posts array may be empty or have pending statuses)
  console.log('Post is still being processed');
  return { status: 'pending', succeeded: [], failed: [] };
}

// Usage
const result = await checkPostGroupStatus('664f1a2b3c4d5e6f7a8b9c0d');
if (result.failed.length > 0) {
  // Take action: notify team, retry failed platforms, etc.
}
```

**Python (requests)**

```python
import requests

API_URL = 'https://api.publora.com/api/v1'
HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
}


def check_post_group_status(post_group_id):
    response = requests.get(
        f'{API_URL}/get-post/{post_group_id}',
        headers=HEADERS
    )

    post_group = response.json()
    posts = post_group.get('posts', [])

    # Determine overall status from individual posts
    statuses = [p['status'] for p in posts]
    all_published = all(s == 'published' for s in statuses)
    all_failed = all(s == 'failed' for s in statuses)

    if all_published:
        overall_status = 'published'
    elif all_failed:
        overall_status = 'failed'
    elif posts:
        overall_status = 'partially_published'
    else:
        overall_status = 'pending'

    print(f"Post group {post_group['postGroupId']}: {overall_status}")

    if overall_status == 'partially_published':
        print('Some platforms failed:')

        failed = [p for p in posts if p['status'] == 'failed']
        succeeded = [p for p in posts if p['status'] == 'published']

        print(f"  Succeeded: {len(succeeded)}")
        for p in succeeded:
            print(f"    - {p['platform']}: {p.get('postedId', 'N/A')}")

        print(f"  Failed: {len(failed)}")
        for p in failed:
            print(f"    - {p['platform']}: {p.get('error', 'Unknown error')}")

        return {'status': 'partial', 'succeeded': succeeded, 'failed': failed}

    if overall_status == 'failed':
        print('All platforms failed.')
        for p in posts:
            print(f"  - {p['platform']}: {p.get('error', 'Unknown error')}")
        return {'status': 'failed', 'succeeded': [], 'failed': posts}

    if overall_status == 'published':
        print('All platforms succeeded.')
        return {'status': 'success', 'succeeded': posts, 'failed': []}

    print(f'Post is still being processed')
    return {'status': 'pending', 'succeeded': [], 'failed': []}


# Usage
result = check_post_group_status('664f1a2b3c4d5e6f7a8b9c0d')
if result['failed']:
    # Take action: notify team, retry failed platforms, etc.
    pass
```

## Best Practices

1. **Always check the HTTP status code.** Do not assume every response is successful. Parse the error body for details.

2. **Only retry on 5xx errors and network failures.** Client errors (4xx) indicate a problem with your request that retrying will not fix.

3. **Use exponential backoff for retries.** Start at 1 second and double with each attempt. Cap at 3-5 retries to avoid infinite loops.

4. **Monitor for partial failures.** A `200` response when creating a post does not guarantee all platforms will succeed. Poll the post group status after the scheduled time to detect partial failures.

5. **Log error responses in full.** When debugging, log the entire response body, not just the error message. Additional fields may provide context.

6. **Handle `403` errors gracefully in your UI.** If a user hits a plan limit (returned as a structured `LimitExceededError`), parse the `metric`, `limit`, `used`, and `remaining` fields to show actionable guidance rather than a raw error.

7. **Validate inputs before sending.** Check that `scheduledTime` is in the future and in ISO 8601 format, platform IDs follow the `{platform}-{id}` format, and media counts are within limits. This avoids unnecessary `400` errors.

## Common Issues

| Problem | Cause | Solution |
|---|---|---|
| `401` on every request | API key not set or incorrect | Ensure `x-publora-key` header is present with a valid key |
| `403` but key is valid | Account has no active subscription, hit a usage limit, or account is on hold/inactive | Check subscription status, review the structured error response for limit details, or contact support |
| `400` when scheduling | `scheduledTime` is in the past or malformed | Always send a future UTC time as `YYYY-MM-DDTHH:mm:ss.sssZ` |
| `404` when fetching a post | Post group ID is wrong or belongs to another account | Verify the ID and that you are using the correct API key |
| Post group shows `partially_published` | Some platforms failed while others succeeded | Inspect individual platform post statuses for error details |
| TikTok post fails with FPS error | Video frame rate below TikTok's minimum requirement | Re-encode the video with at least 24 FPS before uploading |
| `500` intermittent errors | Temporary server issues | Implement retry logic with exponential backoff |
| Threads post fails with nested thread error | Multi-part nested threads are temporarily disabled pending Meta permissions approval | Keep content under 500 characters or use carousel posts. Contact support@publora.com for updates. |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
