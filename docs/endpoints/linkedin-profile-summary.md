# LinkedIn Profile Summary

Retrieve a summary of your LinkedIn profile analytics including follower counts and aggregated post engagement metrics.

## Endpoint

```
POST https://api.publora.com/api/v1/linkedin-profile-summary
```

## Headers

| Header | Required | Description |
|--------|----------|-------------|
| `x-publora-key` | Yes | Your API key |
| `Content-Type` | Yes | `application/json` |

## Request Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `platformId` | string | Yes | LinkedIn connection ID (format: `linkedin-{id}`) |
| `dateRange` | object | No | Date range filter for metrics |
| `dateRange.start` | object | Conditional | Start date: `{ year, month, day }` |
| `dateRange.end` | object | Conditional | End date: `{ year, month, day }` |

> **Note:** Both `start` and `end` are required when `dateRange` is provided. You cannot specify only one of them.

## Response

### Success Response

```json
{
  "success": true,
  "profile": {
    "followers": {
      "total": 5000,
      "periodGrowth": 150
    },
    "posts": {
      "totalImpressions": 25000,
      "totalMembersReached": 12500,
      "totalReactions": 500,
      "totalComments": 75,
      "totalReshares": 30
    }
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `profile.followers.total` | integer | Total follower count |
| `profile.followers.periodGrowth` | integer | Follower growth during date range (only present when `dateRange` is provided) |
| `profile.posts.totalImpressions` | integer or null | Total post impressions (null if LinkedIn returns unavailable) |
| `profile.posts.totalMembersReached` | integer or null | Total unique members who saw your posts (null if LinkedIn returns unavailable) |
| `profile.posts.totalReactions` | integer or null | Total reactions across all posts (null if LinkedIn returns unavailable) |
| `profile.posts.totalComments` | integer or null | Total comments across all posts (null if LinkedIn returns unavailable) |
| `profile.posts.totalReshares` | integer or null | Total reshares across all posts (null if LinkedIn returns unavailable) |

> **Note:** Metric fields become `null` only when LinkedIn explicitly returns `-1` for that metric. If an API call fails entirely, the field defaults to `0` in the profile output (and the failure is reported in the `errors` array).

### Partial Failure Response

When some data was fetched successfully but other requests failed:

```json
{
  "success": false,
  "partialData": true,
  "profile": {
    "followers": {
      "total": 5000
    },
    "posts": {
      "totalImpressions": 25000,
      "totalMembersReached": 0,
      "totalReshares": 30,
      "totalReactions": 500,
      "totalComments": 75
    }
  },
  "errors": [
    { "metric": "followersPeriod", "error": "Failed to fetch follower growth" },
    { "metric": "post_MEMBERS_REACHED", "error": "Failed to fetch members_reached count" }
  ]
}
```

The `partialData: true` flag indicates some metrics are available despite the failure. Check the `errors` array for details on what failed. Each error object contains:
- `metric`: The metric that failed to fetch (uses prefixed format: `followersLifetime`, `followersPeriod`, `post_IMPRESSION`, `post_MEMBERS_REACHED`, `post_REACTION`, `post_COMMENT`, `post_RESHARE`)
- `error`: The error message describing what went wrong. For post metrics, the format is `"Failed to fetch {metric} count"` where `{metric}` uses **lowercase with underscores** (e.g., `"Failed to fetch impression count"`, `"Failed to fetch members_reached count"`, `"Failed to fetch reaction count"`). For follower metrics, it is `"Failed to fetch follower count"` or `"Failed to fetch follower growth"`.

## Examples

### Get full profile summary

#### JavaScript (fetch)

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-profile-summary', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    platformId: 'linkedin-Tz9W5i6ZYG'
  })
});
const data = await response.json();
console.log(`Followers: ${data.profile.followers.total}`);
console.log(`Total Impressions: ${data.profile.posts.totalImpressions}`);
console.log(`Total Engagement: ${data.profile.posts.totalReactions + data.profile.posts.totalComments}`);
```

#### Python (requests)

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/linkedin-profile-summary',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'platformId': 'linkedin-Tz9W5i6ZYG'
    }
)
data = response.json()
profile = data['profile']
print(f"Followers: {profile['followers']['total']}")
print(f"Total Impressions: {profile['posts']['totalImpressions']}")
print(f"Total Reactions: {profile['posts']['totalReactions']}")
```

#### cURL

```bash
curl -X POST https://api.publora.com/api/v1/linkedin-profile-summary \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "platformId": "linkedin-Tz9W5i6ZYG"
  }'
```

### Get profile summary with date range

Track follower growth and metrics for a specific time period:

#### JavaScript (fetch)

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-profile-summary', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    platformId: 'linkedin-Tz9W5i6ZYG',
    dateRange: {
      start: { year: 2026, month: 1, day: 1 },
      end: { year: 2026, month: 1, day: 31 }
    }
  })
});
const data = await response.json();
console.log(`Follower growth in January: ${data.profile.followers.periodGrowth}`);
```

#### Python (requests)

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/linkedin-profile-summary',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'platformId': 'linkedin-Tz9W5i6ZYG',
        'dateRange': {
            'start': {'year': 2026, 'month': 1, 'day': 1},
            'end': {'year': 2026, 'month': 1, 'day': 31}
        }
    }
)
data = response.json()
print(f"Follower growth: {data['profile']['followers']['periodGrowth']}")
```

#### cURL

```bash
curl -X POST https://api.publora.com/api/v1/linkedin-profile-summary \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "platformId": "linkedin-Tz9W5i6ZYG",
    "dateRange": {
      "start": { "year": 2026, "month": 1, "day": 1 },
      "end": { "year": 2026, "month": 1, "day": 31 }
    }
  }'
```

### Handle partial failures

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-profile-summary', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    platformId: 'linkedin-Tz9W5i6ZYG'
  })
});
const data = await response.json();

if (data.partialData) {
  console.warn('Some data could not be fetched:', data.errors);
  // Still use available data
  if (data.profile.posts) {
    console.log(`Impressions: ${data.profile.posts.totalImpressions}`);
  }
} else if (data.success) {
  console.log(`Full profile data received`);
}
```

## Errors

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"platformId is required"` | Missing platformId in request body |
| 400 | `"platformId must be a string"` | platformId is not a string type |
| 400 | `"Invalid platformId"` | platformId format is invalid |
| 400 | `"dateRange must be an object"` | dateRange parameter was provided but is not an object type |
| 400 | `"dateRange must have both 'start' and 'end' objects"` | dateRange provided but missing start or end |
| 400 | `"dateRange.start and dateRange.end must have: year, month, day"` | dateRange start/end missing required fields |
| 401 | `"API key is required"` | No `x-publora-key` header provided |
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 401 | `"Invalid API key owner"` | API key owner user not found in database |
| 403 | `"API access is not enabled for this account"` | Account's plan does not include API access |
| 404 | `"LinkedIn connection not found"` | No LinkedIn account with that platformId |
| 500 | `"Failed to fetch LinkedIn profile summary"` | Server error |
| 502 | `"Failed to fetch LinkedIn profile data"` | Unable to fetch any data from LinkedIn |

The 502 error response includes an `errors` array with details on each failed metric:

```json
{
  "success": false,
  "error": "Failed to fetch LinkedIn profile data",
  "errors": [
    { "metric": "followersLifetime", "error": "Failed to fetch follower count" },
    { "metric": "post_IMPRESSION", "error": "Failed to fetch impression count" }
  ]
}
```

> **Note:** Error status codes from the LinkedIn API may be forwarded directly (e.g., 403, 429), so you may receive error codes other than those listed above.


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) -- the best AI service for authentic thought leadership at scale.*
