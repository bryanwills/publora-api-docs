# How to Sync and Search Your X Bookmarks with Publora

Save, search, filter, and export your X (Twitter) bookmarks using the Publora API. Media (images, videos) is automatically downloaded and stored for permanent access.

> **Beta Feature** — X Bookmarks is currently available on Premium accounts only. Contact support@publora.com or ask your admin to enable it.

## Overview

Publora connects to your X account via OAuth2, syncs your bookmarks, and provides a full API for searching, filtering, and exporting them. Media attachments are automatically downloaded to cloud storage so they remain accessible even if the original tweet is deleted.

## Getting Started

### 1. Enable X Bookmarks

Ask your admin to enable the feature in the Admin Panel (Users > Edit > X Bookmarks Enabled checkbox), or contact support@publora.com.

### 2. Connect Your X Account

Visit your Publora dashboard and go to **Inbox** to connect your X account via OAuth2. This grants read-only access to your bookmarks.

### 3. Sync Bookmarks

Once connected, use the API to sync your bookmarks:

```bash
curl -X POST "https://api.publora.com/api/v1/x-bookmarks/sync" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{"maxTweets": 100}'
```

## API Reference

**Base URL:** `https://api.publora.com/api/v1/x-bookmarks`

**Authentication:** Include your API key in every request:
```
x-publora-key: YOUR_API_KEY
```

## Endpoints

### Sync Bookmarks

```
POST /api/v1/x-bookmarks/sync
```

Fetches bookmarks from X and stores them. Continues from where the last sync left off.

**Request Body (optional):**
```json
{
  "maxTweets": 100
}
```

**Response:**
```json
{
  "success": true,
  "synced": {
    "inserted": 12,
    "updated": 38,
    "total": 50
  },
  "hasMore": true,
  "balance": {
    "used": 162,
    "limit": 2000,
    "remaining": 1838,
    "isUnlimited": false
  }
}
```

**Rate limit:** 1 sync per 60 seconds.

### List Bookmarks

```
GET /api/v1/x-bookmarks?page=1&limit=20
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number (max 500) |
| `limit` | number | 20 | Items per page (max 100) |
| `q` | string | — | Full-text search |
| `author` | string | — | Filter by author username (substring) |
| `from` | ISO date | — | Tweets created after this date |
| `to` | ISO date | — | Tweets created before this date |
| `has_media` | string | — | `true` or `false` |
| `media_type` | string | — | `photo`, `video`, or `animated_gif` |
| `sort` | string | `tweet_date` | `tweet_date`, `synced_at`, `likes`, or `retweets` |

**Response:**
```json
{
  "success": true,
  "bookmarks": [
    {
      "tweetId": "1234567890",
      "text": "This is an interesting tweet about AI...",
      "authorUsername": "johndoe",
      "authorName": "John Doe",
      "tweetCreatedAt": "2026-03-20T15:30:00.000Z",
      "publicMetrics": {
        "likeCount": 542,
        "retweetCount": 89,
        "replyCount": 23,
        "quoteCount": 12
      },
      "media": [
        {
          "mediaKey": "3_1234567890",
          "type": "photo",
          "url": "https://pbs.twimg.com/media/...",
          "s3Url": "https://cdn.publora.com/x-bookmarks/..."
        }
      ],
      "referencedTweets": [
        {
          "type": "quoted",
          "id": "9876543210",
          "content": {
            "text": "Original tweet being quoted...",
            "authorUsername": "originalauthor"
          }
        }
      ]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 342,
    "totalPages": 18,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

**Note:** When using text search (`q`), results are sorted by relevance. You cannot combine `q` with a custom `sort` or with `has_media=false`.

### Get Single Bookmark

```
GET /api/v1/x-bookmarks/:id
```

Accepts a tweet ID or MongoDB ObjectId.

### Export Bookmarks

```
GET /api/v1/x-bookmarks/export?q=ai&has_media=true&limit=100
```

Same filters as List. Returns up to 200 bookmarks with internal tracking fields stripped.

**Rate limit:** 5 exports per 15 minutes.

### Get Balance

```
GET /api/v1/x-bookmarks/balance
```

Returns monthly usage and remaining quota.

**Response:**
```json
{
  "success": true,
  "balance": {
    "used": 150,
    "limit": 2000,
    "remaining": 1850,
    "isUnlimited": false,
    "periodStart": "2026-03-01T00:00:00.000Z",
    "periodEnd": "2026-04-01T00:00:00.000Z"
  }
}
```

### Get/Update Settings

```
GET /api/v1/x-bookmarks/user/settings
PUT /api/v1/x-bookmarks/user/settings
```

**PUT Request Body:**
```json
{
  "downloadImages": true,
  "downloadVideos": false,
  "fetchReferencedTweets": true
}
```

All fields are optional booleans. Defaults: all `true`.

### Get Media Download Stats

```
GET /api/v1/x-bookmarks/user/media-stats
```

**Response:**
```json
{
  "success": true,
  "mediaStats": {
    "pending": 5,
    "downloading": 1,
    "completed": 230,
    "failed": 3,
    "skipped": 12,
    "total": 251
  }
}
```

### Get Sync History

```
GET /api/v1/x-bookmarks/sync/history?page=1&limit=20
```

**Response:**
```json
{
  "success": true,
  "logs": [
    {
      "syncedAt": "2026-03-26T12:00:00.000Z",
      "status": "success",
      "inserted": 5,
      "updated": 15,
      "total": 20,
      "hasMore": false,
      "durationMs": 2340,
      "trigger": "scheduled"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 48,
    "totalPages": 3
  }
}
```

`trigger` values: `manual`, `api`, or `scheduled`.

## Complete JavaScript Implementation

```javascript
const PUBLORA_API_KEY = process.env.PUBLORA_API_KEY;
const BASE_URL = 'https://api.publora.com/api/v1/x-bookmarks';

/**
 * Sync bookmarks from X
 * @param {number} maxTweets - Max bookmarks to fetch (1-100)
 * @returns {Promise<Object>} Sync results with balance
 */
async function syncBookmarks(maxTweets = 100) {
  const response = await fetch(`${BASE_URL}/sync`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': PUBLORA_API_KEY
    },
    body: JSON.stringify({ maxTweets })
  });

  const data = await response.json();

  if (!response.ok) {
    const error = new Error(data.error || `HTTP ${response.status}`);
    error.status = response.status;
    error.code = data.code;

    switch (response.status) {
      case 429:
        error.message = `Rate limited. Retry after ${data.retryAfter}s`;
        break;
      case 403:
        error.message = `Monthly limit reached: ${data.used}/${data.limit}`;
        break;
      case 401:
        error.message = 'X authentication expired. Reconnect your account.';
        break;
    }

    throw error;
  }

  return data;
}

/**
 * Search bookmarks with filters
 * @param {Object} filters - Search and filter parameters
 * @returns {Promise<Object>} Bookmarks with pagination
 */
async function searchBookmarks(filters = {}) {
  const params = new URLSearchParams();
  for (const [key, value] of Object.entries(filters)) {
    if (value !== undefined && value !== null) {
      params.set(key, String(value));
    }
  }

  const response = await fetch(`${BASE_URL}?${params}`, {
    headers: { 'x-publora-key': PUBLORA_API_KEY }
  });

  const data = await response.json();
  if (!response.ok) {
    throw new Error(data.error || `HTTP ${response.status}`);
  }

  return data;
}

/**
 * Export bookmarks as JSON
 * @param {Object} filters - Same filters as searchBookmarks
 * @returns {Promise<Object>} Exported bookmarks (max 200)
 */
async function exportBookmarks(filters = {}) {
  const params = new URLSearchParams();
  for (const [key, value] of Object.entries(filters)) {
    if (value !== undefined && value !== null) {
      params.set(key, String(value));
    }
  }

  const response = await fetch(`${BASE_URL}/export?${params}`, {
    headers: { 'x-publora-key': PUBLORA_API_KEY }
  });

  const data = await response.json();
  if (!response.ok) {
    throw new Error(data.error || `HTTP ${response.status}`);
  }

  return data;
}

// ============ USAGE EXAMPLES ============

// Example 1: Sync latest bookmarks
async function syncLatest() {
  try {
    const result = await syncBookmarks(100);
    console.log(`Synced: ${result.synced.inserted} new, ${result.synced.updated} updated`);
    console.log(`Balance: ${result.balance.used}/${result.balance.limit}`);
    if (result.hasMore) {
      console.log('More bookmarks available — sync again to continue');
    }
    return result;
  } catch (error) {
    console.error('Sync failed:', error.message);
    throw error;
  }
}

// Example 2: Search bookmarks by keyword
async function searchByKeyword(keyword) {
  try {
    const result = await searchBookmarks({ q: keyword, limit: 50 });
    console.log(`Found ${result.pagination.totalItems} bookmarks matching "${keyword}"`);
    for (const b of result.bookmarks) {
      console.log(`  @${b.authorUsername}: ${b.text.substring(0, 80)}...`);
    }
    return result;
  } catch (error) {
    console.error('Search failed:', error.message);
    throw error;
  }
}

// Example 3: Get bookmarks with videos, sorted by likes
async function getPopularVideos() {
  try {
    const result = await searchBookmarks({
      has_media: 'true',
      media_type: 'video',
      sort: 'likes',
      limit: 20
    });
    console.log(`Found ${result.pagination.totalItems} video bookmarks`);
    return result;
  } catch (error) {
    console.error('Failed:', error.message);
    throw error;
  }
}

// Example 4: Export bookmarks from a date range
async function exportDateRange(from, to) {
  try {
    const result = await exportBookmarks({ from, to, limit: 200 });
    console.log(`Exported ${result.count} bookmarks`);
    return result.bookmarks;
  } catch (error) {
    console.error('Export failed:', error.message);
    throw error;
  }
}

// Example 5: Full sync loop (fetch all bookmarks)
async function fullSync() {
  let totalNew = 0;
  let totalUpdated = 0;
  let hasMore = true;

  while (hasMore) {
    const result = await syncBookmarks(100);
    totalNew += result.synced.inserted;
    totalUpdated += result.synced.updated;
    hasMore = result.hasMore;

    if (hasMore) {
      // Respect rate limit: 1 sync per 60 seconds
      console.log(`Synced batch. Waiting 60s before next...`);
      await new Promise(r => setTimeout(r, 60000));
    }
  }

  console.log(`Full sync complete: ${totalNew} new, ${totalUpdated} updated`);
  return { totalNew, totalUpdated };
}

// Run
const syncResult = await syncLatest();
const searchResult = await searchByKeyword('machine learning');
const videos = await getPopularVideos();
const exported = await exportDateRange('2026-01-01', '2026-03-01');
```

## Complete Python Implementation

```python
import os
import time
import requests
from typing import Optional, Dict, Any, List
from urllib.parse import urlencode

PUBLORA_API_KEY = os.environ.get('PUBLORA_API_KEY')
BASE_URL = 'https://api.publora.com/api/v1/x-bookmarks'

HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': PUBLORA_API_KEY,
}


class PubloraBookmarkError(Exception):
    """Custom exception for X Bookmarks API errors."""
    def __init__(self, message: str, status_code: int = None, code: str = None):
        super().__init__(message)
        self.status_code = status_code
        self.code = code


def _handle_error(response: requests.Response):
    """Raise a descriptive error from a failed API response."""
    data = response.json()
    message = data.get('error', f'HTTP {response.status_code}')
    code = data.get('code', '')

    if response.status_code == 429:
        retry = data.get('retryAfter', 60)
        message = f'Rate limited. Retry after {retry}s'
    elif response.status_code == 403:
        message = f'Monthly limit reached: {data.get("used")}/{data.get("limit")}'
    elif response.status_code == 401:
        message = 'X authentication expired. Reconnect your account.'

    raise PubloraBookmarkError(message, response.status_code, code)


def sync_bookmarks(max_tweets: int = 100) -> Dict[str, Any]:
    """
    Sync bookmarks from X.

    Args:
        max_tweets: Max bookmarks to fetch per request (1-100).

    Returns:
        dict with synced counts, hasMore flag, and balance.
    """
    response = requests.post(
        f'{BASE_URL}/sync',
        headers=HEADERS,
        json={'maxTweets': max_tweets},
    )
    if not response.ok:
        _handle_error(response)
    return response.json()


def search_bookmarks(**filters) -> Dict[str, Any]:
    """
    Search bookmarks with filters.

    Keyword Args:
        q: Full-text search query.
        author: Filter by author username (substring).
        from_date: Tweets created after (ISO date). Pass as 'from'.
        to_date: Tweets created before (ISO date). Pass as 'to'.
        has_media: 'true' or 'false'.
        media_type: 'photo', 'video', or 'animated_gif'.
        sort: 'tweet_date', 'synced_at', 'likes', or 'retweets'.
        page: Page number.
        limit: Items per page (max 100).

    Returns:
        dict with bookmarks list and pagination.
    """
    params = {k: v for k, v in filters.items() if v is not None}
    response = requests.get(
        f'{BASE_URL}?{urlencode(params)}',
        headers=HEADERS,
    )
    if not response.ok:
        _handle_error(response)
    return response.json()


def export_bookmarks(**filters) -> Dict[str, Any]:
    """
    Export bookmarks as JSON (max 200). Same filters as search_bookmarks.
    """
    params = {k: v for k, v in filters.items() if v is not None}
    response = requests.get(
        f'{BASE_URL}/export?{urlencode(params)}',
        headers=HEADERS,
    )
    if not response.ok:
        _handle_error(response)
    return response.json()


def get_balance() -> Dict[str, Any]:
    """Get monthly usage and remaining quota."""
    response = requests.get(f'{BASE_URL}/balance', headers=HEADERS)
    if not response.ok:
        _handle_error(response)
    return response.json()


def get_media_stats() -> Dict[str, Any]:
    """Get media download statistics."""
    response = requests.get(f'{BASE_URL}/user/media-stats', headers=HEADERS)
    if not response.ok:
        _handle_error(response)
    return response.json()


# ============ USAGE EXAMPLES ============

def sync_latest():
    """Sync the latest batch of bookmarks."""
    try:
        result = sync_bookmarks(100)
        synced = result['synced']
        balance = result['balance']
        print(f"Synced: {synced['inserted']} new, {synced['updated']} updated")
        print(f"Balance: {balance['used']}/{balance['limit']}")
        if result['hasMore']:
            print("More bookmarks available — sync again to continue")
        return result
    except PubloraBookmarkError as e:
        print(f"Sync failed: {e}")
        raise


def search_by_keyword(keyword: str):
    """Search bookmarks by keyword."""
    try:
        result = search_bookmarks(q=keyword, limit=50)
        total = result['pagination']['totalItems']
        print(f'Found {total} bookmarks matching "{keyword}"')
        for b in result['bookmarks']:
            print(f"  @{b['authorUsername']}: {b['text'][:80]}...")
        return result
    except PubloraBookmarkError as e:
        print(f"Search failed: {e}")
        raise


def get_popular_videos():
    """Get bookmarks with videos, sorted by likes."""
    try:
        result = search_bookmarks(has_media='true', media_type='video', sort='likes', limit=20)
        print(f"Found {result['pagination']['totalItems']} video bookmarks")
        return result
    except PubloraBookmarkError as e:
        print(f"Failed: {e}")
        raise


def export_date_range(from_date: str, to_date: str) -> List[Dict]:
    """Export bookmarks from a date range."""
    try:
        # 'from' is a Python keyword, so we pass it via dict unpacking
        result = export_bookmarks(**{'from': from_date, 'to': to_date, 'limit': 200})
        print(f"Exported {result['count']} bookmarks")
        return result['bookmarks']
    except PubloraBookmarkError as e:
        print(f"Export failed: {e}")
        raise


def full_sync():
    """Sync all bookmarks (respects rate limits)."""
    total_new = 0
    total_updated = 0
    has_more = True

    while has_more:
        result = sync_bookmarks(100)
        total_new += result['synced']['inserted']
        total_updated += result['synced']['updated']
        has_more = result['hasMore']

        if has_more:
            print("Synced batch. Waiting 60s before next...")
            time.sleep(60)

    print(f"Full sync complete: {total_new} new, {total_updated} updated")
    return {'total_new': total_new, 'total_updated': total_updated}


if __name__ == '__main__':
    sync_latest()
    search_by_keyword('machine learning')
    get_popular_videos()
    export_date_range('2026-01-01', '2026-03-01')
```

## cURL Examples

### Sync bookmarks

```bash
curl -X POST "https://api.publora.com/api/v1/x-bookmarks/sync" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{"maxTweets": 100}'
```

### Search by keyword

```bash
curl "https://api.publora.com/api/v1/x-bookmarks?q=machine+learning&limit=20" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Filter by author and date range

```bash
curl "https://api.publora.com/api/v1/x-bookmarks?author=elonmusk&from=2026-01-01&to=2026-03-01" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Get video bookmarks sorted by likes

```bash
curl "https://api.publora.com/api/v1/x-bookmarks?has_media=true&media_type=video&sort=likes" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Export bookmarks

```bash
curl "https://api.publora.com/api/v1/x-bookmarks/export?has_media=true&limit=200" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Check balance

```bash
curl "https://api.publora.com/api/v1/x-bookmarks/balance" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Update settings

```bash
curl -X PUT "https://api.publora.com/api/v1/x-bookmarks/user/settings" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{"downloadImages": true, "downloadVideos": false}'
```

### Get media download stats

```bash
curl "https://api.publora.com/api/v1/x-bookmarks/user/media-stats" \
  -H "x-publora-key: YOUR_API_KEY"
```

### Get sync history

```bash
curl "https://api.publora.com/api/v1/x-bookmarks/sync/history?limit=10" \
  -H "x-publora-key: YOUR_API_KEY"
```

## Important Notes

### Beta Access

X Bookmarks is a beta feature available on Premium accounts only. To get access:
- Contact support@publora.com, or
- Ask your workspace admin to enable it in Admin Panel > Users > Edit

### Monthly Limits

- New bookmarks count against your monthly quota (default: 2000 for Premium)
- Re-syncing existing bookmarks does **not** consume quota — only new inserts count
- Check your balance with `GET /balance` before large syncs

### Auto-Sync

Publora automatically syncs the latest 20 bookmarks for all connected users 3 times per day. Engagement metrics (likes, retweets, replies) are updated on every sync.

### Media Downloads

Images and videos are automatically downloaded to cloud storage by a background process. Check progress with the media stats endpoint. Downloads can be controlled via settings:
- `downloadImages` — photo attachments (default: true)
- `downloadVideos` — video and GIF attachments (default: true)
- `fetchReferencedTweets` — quoted/replied tweet content and media (default: true)

### Rate Limits

| Endpoint | Limit |
|----------|-------|
| Sync | 1 per 60 seconds |
| Export | 5 per 15 minutes |
| All other endpoints | Standard API rate limits |

## Error Reference

| Status | Code | Cause | Solution |
|--------|------|-------|----------|
| 400 | `NOT_CONNECTED` | X account not connected | Connect via dashboard |
| 400 | `CONFLICTING_FILTERS` | Incompatible filter combo | e.g., `has_media=false` + `q` |
| 400 | `INVALID_PARAM_TYPE` | Wrong parameter type | Ensure params are strings |
| 400 | `SEARCH_QUERY_TOO_LONG` | Query exceeds 200 chars | Shorten search query |
| 401 | `AUTH_EXPIRED` | X OAuth token expired | Reconnect X account |
| 403 | `X_BOOKMARKS_LIMIT_REACHED` | Monthly quota exhausted | Wait for next billing period |
| 409 | `SYNC_IN_PROGRESS` | Another sync is running | Wait and retry |
| 429 | `SYNC_RATE_LIMITED` | Sync too frequent | Wait 60 seconds |
| 429 | `EXPORT_RATE_LIMITED` | Too many exports | Wait 15 minutes |
| 429 | `RATE_LIMITED` | X API rate limit | Wait and retry |

## Related Guides

- [LinkedIn Analytics](/docs/guides/linkedin-analytics.md) - Get engagement metrics for LinkedIn posts
