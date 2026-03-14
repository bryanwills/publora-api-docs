# LinkedIn Feed Retrieval API (Coming Soon)

Retrieve your LinkedIn posts, comments, and reactions via the Publora API.

> **Status: NOT YET AVAILABLE** - These endpoints require the `r_member_social` permission, which is **RESTRICTED** and requires LinkedIn approval. The implementation is ready and will be enabled once LinkedIn approves the permission for Publora. Contact LinkedIn Partner Support to request access.

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/linkedin-posts` | POST | Get posts authored by your account |
| `/api/v1/linkedin-post-comments` | POST | Get comments on a specific post |
| `/api/v1/linkedin-post-reactions` | POST | Get reactions on a specific post |

---

## Get LinkedIn Posts

Retrieve posts authored by a connected LinkedIn account.

### Request

```http
POST /api/v1/linkedin-posts
x-publora-key: sk_YOUR_API_KEY
Content-Type: application/json

{
  "platformId": "linkedin-ABC123",
  "start": 0,
  "count": 10,
  "sortBy": "LAST_MODIFIED"
}
```

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `platformId` | string | Yes | - | LinkedIn connection ID (e.g., `linkedin-ABC123`) |
| `start` | number | No | 0 | Starting index for pagination |
| `count` | number | No | 10 | Number of posts (max: 100) |
| `sortBy` | string | No | LAST_MODIFIED | Sort order: `LAST_MODIFIED` or `CREATED` |

### Response

```json
{
  "success": true,
  "posts": [
    {
      "id": "urn:li:share:7123456789",
      "commentary": "Excited to share our latest product update!",
      "publishedAt": 1709294400000,
      "visibility": "PUBLIC",
      "lifecycleState": "PUBLISHED",
      "author": "urn:li:person:ABC123",
      "distribution": {
        "feedDistribution": "MAIN_FEED"
      },
      "content": {
        "media": []
      }
    }
  ],
  "pagination": {
    "start": 0,
    "count": 10,
    "total": 45
  },
  "cached": false
}
```

### cURL Example

```bash
curl -X POST "https://api.publora.com/api/v1/linkedin-posts" \
  -H "x-publora-key: sk_YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "platformId": "linkedin-ABC123",
    "count": 20,
    "sortBy": "LAST_MODIFIED"
  }'
```

### Python Example

```python
import requests

response = requests.post(
    "https://api.publora.com/api/v1/linkedin-posts",
    headers={"x-publora-key": "sk_YOUR_API_KEY"},
    json={
        "platformId": "linkedin-ABC123",
        "count": 20,
        "sortBy": "LAST_MODIFIED"
    }
)

data = response.json()
for post in data["posts"]:
    print(f"Post: {post['commentary'][:50]}...")
    print(f"Published: {post['publishedAt']}")
    print("---")
```

---

## Get Post Comments

Retrieve comments on a specific LinkedIn post.

### Request

```http
POST /api/v1/linkedin-post-comments
x-publora-key: sk_YOUR_API_KEY
Content-Type: application/json

{
  "postedId": "urn:li:share:7123456789",
  "platformId": "linkedin-ABC123",
  "start": 0,
  "count": 20
}
```

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `postedId` | string | Yes | - | LinkedIn post URN |
| `platformId` | string | Yes | - | LinkedIn connection ID |
| `start` | number | No | 0 | Starting index for pagination |
| `count` | number | No | 20 | Number of comments (max: 100) |

### Response

```json
{
  "success": true,
  "comments": [
    {
      "id": "6636062862760562688",
      "commentUrn": "urn:li:comment:(urn:li:activity:123,6636062862760562688)",
      "actor": "urn:li:person:XYZ789",
      "message": "Great insights! Thanks for sharing.",
      "created": {
        "time": 1582160678569
      },
      "lastModified": {
        "time": 1582160678569
      },
      "likesSummary": {
        "totalLikes": 5
      },
      "commentsCount": 2
    }
  ],
  "pagination": {
    "start": 0,
    "count": 20,
    "total": 89
  },
  "cached": false
}
```

### cURL Example

```bash
curl -X POST "https://api.publora.com/api/v1/linkedin-post-comments" \
  -H "x-publora-key: sk_YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "postedId": "urn:li:share:7123456789",
    "platformId": "linkedin-ABC123",
    "count": 50
  }'
```

### Python Example

```python
import requests

response = requests.post(
    "https://api.publora.com/api/v1/linkedin-post-comments",
    headers={"x-publora-key": "sk_YOUR_API_KEY"},
    json={
        "postedId": "urn:li:share:7123456789",
        "platformId": "linkedin-ABC123",
        "count": 50
    }
)

data = response.json()
for comment in data["comments"]:
    print(f"Comment by {comment['actor']}: {comment['message']}")
    print(f"Likes: {comment['likesSummary']['totalLikes']}")
    print("---")
```

---

## Get Post Reactions

Retrieve reactions on a specific LinkedIn post.

### Request

```http
POST /api/v1/linkedin-post-reactions
x-publora-key: sk_YOUR_API_KEY
Content-Type: application/json

{
  "postedId": "urn:li:share:7123456789",
  "platformId": "linkedin-ABC123",
  "start": 0,
  "count": 50
}
```

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `postedId` | string | Yes | - | LinkedIn post URN |
| `platformId` | string | Yes | - | LinkedIn connection ID |
| `start` | number | No | 0 | Starting index for pagination |
| `count` | number | No | 50 | Number of reactions (max: 100) |

### Response

```json
{
  "success": true,
  "reactions": [
    {
      "id": "urn:li:reaction:(urn:li:person:XYZ789,urn:li:activity:123)",
      "reactionType": "LIKE",
      "actor": "urn:li:person:XYZ789",
      "created": {
        "time": 1686183251857
      }
    },
    {
      "id": "urn:li:reaction:(urn:li:person:ABC456,urn:li:activity:123)",
      "reactionType": "PRAISE",
      "actor": "urn:li:person:ABC456",
      "created": {
        "time": 1686183300000
      }
    }
  ],
  "pagination": {
    "start": 0,
    "count": 50,
    "total": 456
  },
  "cached": false
}
```

### Reaction Types

| Type | Description | Emoji |
|------|-------------|-------|
| `LIKE` | Standard thumbs up | 👍 |
| `PRAISE` | Clapping hands | 👏 |
| `EMPATHY` | Heart/love | ❤️ |
| `INTEREST` | Lightbulb (insightful) | 💡 |
| `APPRECIATION` | Thank you | 🙏 |
| `ENTERTAINMENT` | Funny/laughing | 😂 |

### cURL Example

```bash
curl -X POST "https://api.publora.com/api/v1/linkedin-post-reactions" \
  -H "x-publora-key: sk_YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "postedId": "urn:li:share:7123456789",
    "platformId": "linkedin-ABC123",
    "count": 100
  }'
```

### Python Example

```python
import requests
from collections import Counter

response = requests.post(
    "https://api.publora.com/api/v1/linkedin-post-reactions",
    headers={"x-publora-key": "sk_YOUR_API_KEY"},
    json={
        "postedId": "urn:li:share:7123456789",
        "platformId": "linkedin-ABC123",
        "count": 100
    }
)

data = response.json()

# Count reactions by type
reaction_counts = Counter(r["reactionType"] for r in data["reactions"])
print("Reaction breakdown:")
for reaction_type, count in reaction_counts.most_common():
    print(f"  {reaction_type}: {count}")
```

---

## Caching

All feed retrieval endpoints are cached to reduce API calls:

| Endpoint | Cache TTL |
|----------|-----------|
| Posts | 15 minutes |
| Comments | 10 minutes |
| Reactions | 10 minutes |

The `cached` field in responses indicates if data was served from cache.

---

## Error Responses

### Permission Denied (403)

```json
{
  "error": "Access denied. The r_member_social permission may not be approved for this application."
}
```

This error indicates the LinkedIn `r_member_social` permission is not approved. Contact LinkedIn Partner Support to request access.

### Connection Not Found (404)

```json
{
  "error": "LinkedIn connection not found"
}
```

Verify the `platformId` matches a connected LinkedIn account.

### Invalid URN (400)

```json
{
  "error": "postedId must be a valid LinkedIn URN"
}
```

Ensure the post URN format is correct (e.g., `urn:li:share:123456` or `urn:li:ugcPost:123456`).

---

## Complete Workflow Example

```python
import requests

API_KEY = "sk_YOUR_API_KEY"
BASE_URL = "https://api.publora.com/api/v1"
HEADERS = {"x-publora-key": API_KEY}

def get_linkedin_platform_id():
    """Get the LinkedIn platform connection ID."""
    response = requests.get(
        f"{BASE_URL}/platform-connections",
        headers=HEADERS
    )
    connections = response.json()["connections"]

    for conn in connections:
        if conn["platformId"].startswith("linkedin-"):
            return conn["platformId"]

    raise ValueError("No LinkedIn connection found")

def analyze_post_engagement(platform_id, post_id):
    """Analyze engagement on a specific post."""
    # Get comments
    comments_response = requests.post(
        f"{BASE_URL}/linkedin-post-comments",
        headers=HEADERS,
        json={"postedId": post_id, "platformId": platform_id, "count": 100}
    )
    comments = comments_response.json()

    # Get reactions
    reactions_response = requests.post(
        f"{BASE_URL}/linkedin-post-reactions",
        headers=HEADERS,
        json={"postedId": post_id, "platformId": platform_id, "count": 100}
    )
    reactions = reactions_response.json()

    return {
        "total_comments": comments["pagination"]["total"],
        "total_reactions": reactions["pagination"]["total"],
        "reaction_breakdown": Counter(r["reactionType"] for r in reactions["reactions"])
    }

# Main workflow
platform_id = get_linkedin_platform_id()

# Get recent posts
posts_response = requests.post(
    f"{BASE_URL}/linkedin-posts",
    headers=HEADERS,
    json={"platformId": platform_id, "count": 5}
)

for post in posts_response.json()["posts"]:
    print(f"\nPost: {post['commentary'][:50]}...")
    engagement = analyze_post_engagement(platform_id, post["id"])
    print(f"  Comments: {engagement['total_comments']}")
    print(f"  Reactions: {engagement['total_reactions']}")
    print(f"  Breakdown: {dict(engagement['reaction_breakdown'])}")
```

---

## MCP Integration

These endpoints are also available as MCP tools:

- `linkedin_posts` - Retrieve your posts
- `linkedin_post_comments` - Retrieve comments on a post
- `linkedin_post_reactions` - Retrieve reactions on a post

See [MCP Tools Reference](/mcp/tools-reference) for details.
