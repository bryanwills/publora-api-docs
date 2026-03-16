# LinkedIn Comments

Create and delete comments on LinkedIn posts programmatically.

## Create Comment

```
POST https://api.publora.com/api/v1/linkedin-comments
```

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `x-publora-key` | Yes | Your API key |
| `Content-Type` | Yes | `application/json` |

### Request Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123`, `urn:li:ugcPost:456`, `urn:li:activity:789`) |
| `message` | string | Yes | The comment text |
| `platformId` | string | Yes | LinkedIn connection ID (format: `linkedin-ABC123`) |
| `parentComment` | string | No | Parent comment URN for threaded replies |

### Response (HTTP 201 Created)

```json
{
  "success": true,
  "comment": {
    "id": "6636062862760562688",
    "commentUrn": "urn:li:comment:(urn:li:activity:7123456789012345678,6636062862760562688)",
    "message": "Great post! Thanks for sharing.",
    "created": {
      "time": 1710345600000
    }
  }
}
```

> **Note:** The `id` field is the numeric comment ID extracted from the LinkedIn API's `x-restli-id` response header. The full comment URN is in the `commentUrn` field. The `comment` object also includes data returned by the LinkedIn API spread into it, so additional fields from the LinkedIn response may be present.

### Examples

#### JavaScript (fetch)

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7123456789012345678',
    message: 'Great post! Thanks for sharing.',
    platformId: 'linkedin-Tz9W5i6ZYG'
  })
});
const data = await response.json();
console.log(data.comment.id);
```

#### Python (requests)

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/linkedin-comments',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'postedId': 'urn:li:share:7123456789012345678',
        'message': 'Great post! Thanks for sharing.',
        'platformId': 'linkedin-Tz9W5i6ZYG'
    }
)
print(response.json()['comment']['id'])
```

#### cURL

```bash
curl -X POST https://api.publora.com/api/v1/linkedin-comments \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:share:7123456789012345678",
    "message": "Great post! Thanks for sharing.",
    "platformId": "linkedin-Tz9W5i6ZYG"
  }'
```

### Reply to a Comment

To create a threaded reply, include the `parentComment` parameter:

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7123456789012345678',
    message: 'Thanks for the kind words!',
    platformId: 'linkedin-Tz9W5i6ZYG',
    parentComment: 'urn:li:comment:(urn:li:activity:7123456789012345678,6636062862760562688)'
  })
});
```

---

## Delete Comment

```
DELETE https://api.publora.com/api/v1/linkedin-comments
```

### Request Body or Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN |
| `commentId` | string | Yes | Comment ID or URN to delete |
| `platformId` | string | Yes | LinkedIn connection ID |

### Response

```json
{
  "success": true,
  "deleted": "6636062862760562688"
}
```

The `deleted` field contains the comment ID that was removed. This can be a numeric ID or a full URN (e.g., `urn:li:comment:(urn:li:activity:123,6636062862760562688)`), depending on what was passed as `commentId`.

### Examples

#### JavaScript (fetch)

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'DELETE',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7123456789012345678',
    commentId: '6636062862760562688',
    platformId: 'linkedin-Tz9W5i6ZYG'
  })
});
```

#### Python (requests)

```python
response = requests.delete(
    'https://api.publora.com/api/v1/linkedin-comments',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'postedId': 'urn:li:share:7123456789012345678',
        'commentId': '6636062862760562688',
        'platformId': 'linkedin-Tz9W5i6ZYG'
    }
)
```

#### cURL

```bash
curl -X DELETE "https://api.publora.com/api/v1/linkedin-comments" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:share:7123456789012345678",
    "commentId": "6636062862760562688",
    "platformId": "linkedin-Tz9W5i6ZYG"
  }'
```

## Errors

### Create Comment Errors (POST)

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"postedId, message, and platformId are required"` | Missing required fields |
| 400 | `"platformId must be a string"` | platformId is not a string type |
| 400 | `"postedId must be a valid LinkedIn URN"` | Not a valid LinkedIn URN format |
| 400 | `"message cannot be empty"` | Message is an empty string |
| 400 | `"message cannot exceed 1250 characters"` | Message exceeds LinkedIn's character limit |
| 400 | `"parentComment must be a valid LinkedIn URN"` | parentComment is not a valid URN format |
| 400 | `"Invalid platformId"` | platformId format is invalid |
| 401 | `"API key is required"` | No `x-publora-key` header provided |
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 401 | `"Invalid API key owner"` | API key owner user not found in database |
| 403 | `"API access is not enabled for this account"` | Account's plan does not include API access |
| 404 | `"LinkedIn connection not found"` | No LinkedIn account with that platformId |
| 500 | `"Failed to create LinkedIn comment"` | Server error while creating comment |

### Delete Comment Errors (DELETE)

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"postedId, commentId, and platformId are required"` | Missing required fields |
| 400 | `"platformId must be a string"` | platformId is not a string type |
| 400 | `"postedId must be a valid LinkedIn URN"` | Not a valid LinkedIn URN format |
| 400 | `"commentId must be a valid LinkedIn URN or numeric ID"` | commentId format is invalid |
| 400 | `"Invalid platformId"` | platformId format is invalid |
| 401 | `"API key is required"` | No `x-publora-key` header provided |
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 401 | `"Invalid API key owner"` | API key owner user not found in database |
| 403 | `"API access is not enabled for this account"` | Account's plan does not include API access |
| 404 | `"LinkedIn connection not found"` | No LinkedIn account with that platformId |
| 500 | `"Failed to delete LinkedIn comment"` | Server error while deleting comment |

> **Note:** The 500 error response includes a `details` field with the underlying error message: `{ "error": "Failed to delete LinkedIn comment", "details": "<error message>" }`.

> **Note:** Error status codes from the LinkedIn API may be forwarded directly (e.g., 403, 429), so you may receive error codes other than those listed above.

## Implementation Notes

- **Create comment response field precedence:** The create comment response constructs the `comment` object by spreading `...response.data` from the LinkedIn API, then explicitly setting `id`, `commentUrn`, and `message`. If LinkedIn ever returns fields named `id`, `commentUrn`, or `message` in its response body, the explicit values will override them. However, if LinkedIn adds new fields with those names in a different format, the spread could briefly surface unexpected data before the override. In practice this is unlikely but worth noting for defensive parsing.

- **Delete comment `commentId` format:** The REST API accepts both a full URN (e.g., `urn:li:comment:(urn:li:activity:123,456)`) and a plain numeric comment ID (e.g., `"6636062862760562688"`). The MCP tool description only mentions URN format, but both formats work identically via the REST API.

---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) -- the best AI service for authentic thought leadership at scale.*
