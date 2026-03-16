# MCP Tools Reference

Complete reference for all 18 active Publora MCP tools with parameters, examples, and code snippets. Three additional tools (`linkedin_posts`, `linkedin_post_comments`, `linkedin_post_reactions`) are temporarily disabled pending LinkedIn approval of the `r_member_social` permission — see [LinkedIn Feed Retrieval Tools](#linkedin-feed-retrieval-tools-coming-soon--requires-linkedin-approval) below.

> **Note:** Most MCP tool responses return the full `{ success, ... }` API response wrapper as shown in the examples below. A few tools — `list_connections`, `list_workspace_users`, and `create_workspace_user` — strip the wrapper and return the inner data directly (e.g., just the `connections` array). The response examples below reflect the actual format returned by each tool.

## Posts Tools

### list_posts

List posts with optional filters for status, platform, and date range.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `status` | string | No | Filter by status: `draft`, `scheduled`, `published`, `failed`, `partially_published` |
| `platform` | string | No | Filter by platform: `twitter`, `linkedin`, `instagram`, `threads`, `tiktok`, `youtube`, `facebook`, `bluesky`, `mastodon`, `telegram` |
| `fromDate` | string | No | Start date (ISO 8601): `2026-02-01T00:00:00Z` |
| `toDate` | string | No | End date (ISO 8601): `2026-02-28T23:59:59Z` |
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Results per page (default: 20, max: 100) |
| `sortBy` | string | No | Sort field: `createdAt`, `updatedAt`, `scheduledTime` (default: createdAt) |
| `sortOrder` | string | No | Sort direction: `asc` or `desc` (default: desc) |

**Example prompts:**

```text
"Show my scheduled posts"
"List all posts from last week"
"What LinkedIn posts are scheduled for next month?"
"Show me failed posts"
"List my drafts"
```

**Python example:**

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def list_scheduled_posts():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List scheduled posts
            result = await session.call_tool("list_posts", {
                "status": "scheduled",
                "limit": 50
            })
            print(result.content[0].text)

asyncio.run(list_scheduled_posts())
```

**Response example:**

```json
{
  "success": true,
  "posts": [
    {
      "postGroupId": "67a1b2c3d4e5f6a7b8c9d0e1",
      "content": "Excited to share our latest update!",
      "status": "scheduled",
      "scheduledTime": "2026-02-20T14:00:00Z",
      "platforms": [
        {
          "platformId": "linkedin-123456",
          "platform": "linkedin",
          "status": "scheduled"
        }
      ],
      "createdAt": "2026-02-19T10:30:00Z",
      "updatedAt": "2026-02-19T10:30:00Z",
      "mediaUrls": []
    }
  ],
  "pagination": {
    "totalItems": 1,
    "totalPages": 1,
    "page": 1,
    "limit": 20,
    "hasNextPage": false,
    "hasPrevPage": false
  }
}
```

---

### create_post

Create and schedule a post to one or more platforms.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `content` | string | Yes | Post text content |
| `platforms` | string[] | Yes | Array of platform connection IDs (from `list_connections`) |
| `scheduledTime` | string | Yes | When to publish (ISO 8601): `2026-03-01T14:00:00Z` |

> **MCP vs REST difference:** `scheduledTime` is required in the MCP schema to ensure AI agents always specify a publish time. In the REST API, this field is optional — omitting it creates a draft post instead.

**Example prompts:**

```text
"Schedule 'Hello world!' to LinkedIn for tomorrow at 9am"
"Post 'We're hiring!' to Twitter and LinkedIn right now"
"Schedule this announcement to all my accounts for Monday"
```

**Python example:**

```python
import asyncio
from datetime import datetime, timedelta
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def schedule_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # First get connections to find platform IDs
            connections = await session.call_tool("list_connections", {})

            # Schedule a post for tomorrow at 9am UTC
            tomorrow_9am = (datetime.utcnow() + timedelta(days=1)).replace(
                hour=9, minute=0, second=0, microsecond=0
            ).isoformat() + "Z"

            result = await session.call_tool("create_post", {
                "content": "Excited to share our latest product update!",
                "platforms": ["linkedin-abc123"],
                "scheduledTime": tomorrow_9am
            })
            print(result.content[0].text)

asyncio.run(schedule_post())
```

**LinkedIn Mentions:**

You can @mention people and companies in LinkedIn posts using this syntax in your content:

```text
@{urn:li:person:4986615|Serge Bulaev}           # Mention a person
@{urn:li:organization:107107343|Acme Corp Inc}  # Mention a company
```

**Example:**
```text
"Great insights from @{urn:li:person:4986615|Serge Bulaev} at @{urn:li:organization:107107343|Creative Content Crafts Inc}!"
```

**Important:** The display name must exactly match the LinkedIn profile name (case-sensitive), including company suffixes like "Inc", "LLC", etc.

**Platform limits:**

| Platform | Characters | Images | Video | Special Features |
|----------|------------|--------|-------|------------------|
| LinkedIn | 3,000 | 20 | 500MB | Documents, @mentions |
| X/Twitter | 280 (25K premium) | 4 | 120s | Auto-threading |
| Instagram | 2,200 | 10 | 3min | Reels supported |
| Threads | 500 | 10 | 5min | Auto-threading |
| TikTok | 2,200 | N/A | 10min | Video-only platform |
| YouTube | 5,000 desc | N/A | 12h | Shorts support |
| Facebook | 63,206 | 10 | 45min | Page posts, Reels |
| Bluesky | 300 | 4 | 3 min / 100 MB | Auto-facet detection |
| Mastodon | 500* | 4 | ~99 MB | Instance-variable |
| Telegram | 4,096 (1,024 captions) | Unlimited | 2GB | Markdown/HTML support |

*Varies by instance

> **Note:** LinkedIn's "20 images" is the multi-image upload limit, not a carousel. Organic carousels on LinkedIn are **not** supported via the API (carousels are only available for sponsored/ad content). To share multi-page content organically, use LinkedIn document (PDF) posts instead.

> **Note:** Multi-threaded nested posts on Threads are temporarily unavailable. Single posts and carousel posts to Threads continue to work normally.

---

### get_post

Get details of a specific post group.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postGroupId` | string | Yes | Post group ID (e.g., `67a1b2c3d4e5f6a7b8c9d0e1`) |

**Example prompts:**

```text
"Show me the details of post 67a1b2c3d4e5f6a7b8c9d0e1"
"What's the status of my last scheduled post?"
"Get info about my recent post"
```

**Python example:**

```python
async def get_post_details():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("get_post", {
                "postGroupId": "67a1b2c3d4e5f6a7b8c9d0e1"
            })
            print(result.content[0].text)
```

---

### update_post

Reschedule or change post status.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postGroupId` | string | Yes | Post group ID |
| `status` | string | No | New status: `draft` or `scheduled` |
| `scheduledTime` | string | No | New scheduled time (ISO 8601) |

**Example prompts:**

```text
"Reschedule post 67a1b2c3d4e5f6a7b8c9d0e1 to Friday at 3pm"
"Change my draft to scheduled"
"Move tomorrow's post to next week"
"Update the post to publish at 10am instead"
```

**Python example:**

```python
async def reschedule_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("update_post", {
                "postGroupId": "67a1b2c3d4e5f6a7b8c9d0e1",
                "scheduledTime": "2026-03-01T15:00:00Z"
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "success": true,
  "message": "Post updated successfully",
  "postGroup": {
    "_id": "67a1b2c3d4e5f6a7b8c9d0e1",
    "status": "scheduled",
    "scheduledTime": "2026-03-01T15:00:00Z"
  }
}
```

> **Note:** Only posts with status `draft` or `scheduled` can be updated. Attempting to update a post in any other status (e.g., `published`, `failed`, `partially_published`) returns a `400` error: `"Cannot update post: post is currently in {status} status"`.

---

### delete_post

Delete a post from all platforms.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postGroupId` | string | Yes | Post group ID to delete |

**Example prompts:**

```text
"Delete post 67a1b2c3d4e5f6a7b8c9d0e1"
"Cancel my scheduled post for tomorrow"
"Remove all my draft posts"
```

**Python example:**

```python
async def delete_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("delete_post", {
                "postGroupId": "67a1b2c3d4e5f6a7b8c9d0e1"
            })
            print(result.content[0].text)
```

---

### get_upload_url

Get a presigned URL to upload media files.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postGroupId` | string | Yes | Post group ID to attach media to |
| `fileName` | string | Yes | File name (e.g., `photo.jpg`) |
| `contentType` | string | Yes | MIME type (e.g., `image/jpeg`, `video/mp4`) |
| `type` | string | Yes | Media type: `image` or `video` |

**Supported formats:**

| Type | Formats |
|------|---------|
| Images | JPEG, PNG, GIF, WebP |
| Videos | MP4, MOV, AVI, WebM |

**Python example:**

```python
import aiohttp

async def upload_image_to_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Get upload URL
            result = await session.call_tool("get_upload_url", {
                "postGroupId": "67a1b2c3d4e5f6a7b8c9d0e1",
                "fileName": "product-photo.jpg",
                "contentType": "image/jpeg",
                "type": "image"
            })

            # Parse the response - contains uploadUrl, fileUrl, and mediaId
            # Response: { "success": true, "uploadUrl": "https://...", "fileUrl": "https://...", "mediaId": "..." }
            import json
            data = json.loads(result.content[0].text)
            upload_url = data["uploadUrl"]

            # Upload file using the presigned URL
            async with aiohttp.ClientSession() as http:
                with open("product-photo.jpg", "rb") as f:
                    await http.put(upload_url, data=f.read())
```

---

## Connections Tool

### list_connections

List all connected social media accounts.

**Parameters:** None

**Example prompts:**

```text
"Show my connected accounts"
"What platforms am I connected to?"
"List my social media accounts"
"Which accounts do I have linked?"
```

**Python example:**

```python
async def list_connections():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("list_connections", {})
            print(result.content[0].text)

asyncio.run(list_connections())
```

**Response example:**

```json
[
  {
    "platformId": "twitter-123456789",
    "username": "@yourcompany",
    "displayName": "Your Company",
    "profileImageUrl": "https://pbs.twimg.com/profile_images/...",
    "profileUrl": null,
    "tokenStatus": "unknown",
    "tokenExpiresIn": null,
    "accessTokenExpiresAt": null,
    "lastSuccessfulPost": null,
    "lastError": null
  },
  {
    "platformId": "linkedin-Tz9W5i6ZYG",
    "username": "Your Company Page",
    "displayName": "Your Company",
    "profileImageUrl": "https://media.licdn.com/...",
    "profileUrl": "https://www.linkedin.com/company/your-company",
    "tokenStatus": "valid",
    "tokenExpiresIn": "82d 4h",
    "accessTokenExpiresAt": "2026-06-15T12:00:00.000Z",
    "lastSuccessfulPost": "2026-03-10T09:30:00.000Z",
    "lastError": null
  }
]
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `platformId` | string | Unique ID for creating posts (e.g., `twitter-123456789`) |
| `username` | string | Platform username or handle |
| `displayName` | string | Display name on the platform |
| `profileImageUrl` | string | Profile image URL |
| `profileUrl` | string/null | URL to the profile on the platform |
| `tokenStatus` | string | Token health: `valid`, `expiring_soon`, `expired`, `unknown` |
| `tokenExpiresIn` | string/null | Human-readable time until expiration (e.g., "7d 3h") |
| `accessTokenExpiresAt` | string/null | ISO 8601 timestamp when the access token expires |
| `lastSuccessfulPost` | string/null | ISO 8601 timestamp of the last successful post via this connection |
| `lastError` | object/null | Last error details: `{ message: string, occurredAt: string }` |

---

## LinkedIn Analytics Tools

### linkedin_post_stats

Get engagement metrics for a specific LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123456`) |
| `platformId` | string | Yes | Platform connection ID |
| `queryTypes` | string[] | No | Metrics to fetch: `IMPRESSION`, `REACTION`, `COMMENT`, `SHARE` (the backend may accept additional types) |

**Example prompts:**

```text
"How did my last LinkedIn post perform?"
"Get impressions for my LinkedIn post"
"Show engagement on my recent LinkedIn update"
"What's the reach of my LinkedIn announcement?"
```

**Python example:**

```python
async def get_linkedin_post_stats():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_post_stats", {
                "postedId": "urn:li:share:7123456789",
                "platformId": "linkedin-abc123",
                "queryTypes": ["IMPRESSION", "REACTION", "COMMENT", "SHARE"]
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "impressions": 1250,
  "uniqueImpressions": 680,
  "reactions": 28,
  "comments": 5,
  "shares": 3,
  "engagementRate": 2.9
}
```

---

### linkedin_account_stats

Get aggregated statistics for your LinkedIn account.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `platformId` | string | Yes | Platform connection ID |
| `queryTypes` | string[] | No | Metrics to fetch |
| `aggregation` | string | No | `DAILY` or `TOTAL` (default: TOTAL) |

**Example prompts:**

```text
"Show my LinkedIn analytics"
"What's my total engagement on LinkedIn?"
"Give me daily LinkedIn stats for this week"
```

**Python example:**

```python
async def get_linkedin_account_stats():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_account_stats", {
                "platformId": "linkedin-abc123",
                "aggregation": "TOTAL"
            })
            print(result.content[0].text)
```

---

### linkedin_followers

Get follower count or growth over time.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `platformId` | string | Yes | Platform connection ID |
| `period` | string | No | `lifetime` or `daily` |
| `dateRange` | object | No | For daily period. Structure: `{start: {year, month, day}, end: {year, month, day}}` |

**Example prompts:**

```text
"How many LinkedIn followers do I have?"
"Show my follower growth this month"
"What's my LinkedIn follower count?"
"Track my follower growth over the last 30 days"
```

**Python example:**

```python
async def get_linkedin_followers():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Lifetime followers
            result = await session.call_tool("linkedin_followers", {
                "platformId": "linkedin-abc123",
                "period": "lifetime"
            })
            print(result.content[0].text)

            # Daily growth
            result = await session.call_tool("linkedin_followers", {
                "platformId": "linkedin-abc123",
                "period": "daily",
                "dateRange": {
                    "start": {"year": 2026, "month": 2, "day": 1},
                    "end": {"year": 2026, "month": 2, "day": 28}
                }
            })
            print(result.content[0].text)
```

---

### linkedin_profile_summary

Get a combined profile overview with followers and stats.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `platformId` | string | Yes | Platform connection ID |
| `dateRange` | object | No | Date range for stats. Structure: `{start: {year, month, day}, end: {year, month, day}}` |

**Example prompts:**

```text
"Give me a summary of my LinkedIn profile"
"How's my LinkedIn doing overall?"
"LinkedIn profile overview"
```

**Python example:**

```python
async def get_linkedin_summary():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_profile_summary", {
                "platformId": "linkedin-abc123"
            })
            print(result.content[0].text)
```

---

### linkedin_create_reaction

React to a LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN |
| `platformId` | string | Yes | Platform connection ID |
| `reactionType` | string | Yes | Reaction type (see below) |

**Reaction types:**

| Type | Description |
|------|-------------|
| `LIKE` | Standard like |
| `PRAISE` | Clapping hands |
| `EMPATHY` | Heart/love |
| `INTEREST` | Lightbulb/insightful |
| `APPRECIATION` | Thank you |
| `ENTERTAINMENT` | Funny/laughing |

**Example prompts:**

```text
"Like this LinkedIn post"
"React with PRAISE to post xyz"
"Add a heart reaction to my colleague's post"
```

**Python example:**

```python
async def react_to_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_create_reaction", {
                "postedId": "urn:li:share:7123456789",
                "platformId": "linkedin-abc123",
                "reactionType": "LIKE"
            })
            print(result.content[0].text)
```

---

### linkedin_delete_reaction

Remove a reaction from a LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN |
| `platformId` | string | Yes | Platform connection ID |

**Example prompts:**

```text
"Remove my reaction from this post"
"Unlike the LinkedIn post"
```

---

## LinkedIn Comment Tools

### linkedin_create_comment

Post a comment on a LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123456` or `urn:li:ugcPost:123456`) |
| `platformId` | string | Yes | Platform connection ID |
| `message` | string | Yes | Comment text (max 1,250 characters) |
| `parentComment` | string | No | Parent comment URN for nested replies |

**Example prompts:**

```text
"Comment 'Great insights!' on this LinkedIn post"
"Post a comment on my latest LinkedIn update"
"Reply to this comment with 'Thanks for sharing!'"
```

**Python example:**

```python
async def comment_on_post():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_create_comment", {
                "postedId": "urn:li:ugcPost:7429953213384187904",
                "platformId": "linkedin-abc123",
                "message": "Great insights! Thanks for sharing."
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "success": true,
  "comment": {
    "id": "7434695495614312448",
    "commentUrn": "urn:li:comment:(urn:li:ugcPost:xxx,7434695495614312448)",
    "message": "Great insights! Thanks for sharing."
  }
}
```

---

### linkedin_delete_comment

Delete a comment from a LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN the comment belongs to |
| `commentId` | string | Yes | Comment URN to delete |
| `platformId` | string | Yes | Platform connection ID |

**Example prompts:**

```text
"Delete my comment from this post"
"Remove the comment I just posted"
```

**Python example:**

```python
async def delete_comment():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_delete_comment", {
                "postedId": "urn:li:ugcPost:7429953213384187904",
                "commentId": "urn:li:comment:(urn:li:ugcPost:xxx,7434695495614312448)",
                "platformId": "linkedin-abc123"
            })
            print(result.content[0].text)
```

---

## LinkedIn Feed Retrieval Tools (Coming Soon — Requires LinkedIn Approval)

> **Status: DISABLED** - These tools are not yet available. They require the `r_member_social` permission, which is **RESTRICTED** and requires LinkedIn approval. The implementation is ready and will be enabled once LinkedIn approves the permission for Publora.

### linkedin_posts

Retrieve posts authored by a LinkedIn account.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `platformId` | string | Yes | Platform connection ID (e.g., `linkedin-XxxYyy`) |
| `start` | number | No | Starting index for pagination (default: 0) |
| `count` | number | No | Number of posts to retrieve (default: 10, max: 100) |
| `sortBy` | string | No | Sort order: `LAST_MODIFIED` or `CREATED` (default: LAST_MODIFIED) |

**Example prompts:**

```text
"Show my recent LinkedIn posts"
"Get my last 10 LinkedIn posts"
"What have I posted on LinkedIn this month?"
"List my LinkedIn content"
```

**Python example:**

```python
async def get_my_linkedin_posts():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_posts", {
                "platformId": "linkedin-abc123",
                "count": 20,
                "sortBy": "LAST_MODIFIED"
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "success": true,
  "posts": [
    {
      "id": "urn:li:share:7123456789",
      "commentary": "Excited to share our latest product update!",
      "publishedAt": 1709294400000,
      "visibility": "PUBLIC",
      "lifecycleState": "PUBLISHED"
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

---

### linkedin_post_comments

Retrieve comments on a specific LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123456` or `urn:li:ugcPost:123456`) |
| `platformId` | string | Yes | Platform connection ID |
| `start` | number | No | Starting index for pagination (default: 0) |
| `count` | number | No | Number of comments to retrieve (default: 20, max: 100) |

**Example prompts:**

```text
"Show comments on my last LinkedIn post"
"Get all comments on this LinkedIn post"
"What are people saying about my post?"
"List comments on post urn:li:share:123456"
```

**Python example:**

```python
async def get_post_comments():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_post_comments", {
                "postedId": "urn:li:ugcPost:7429953213384187904",
                "platformId": "linkedin-abc123",
                "count": 50
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "success": true,
  "comments": [
    {
      "id": "6636062862760562688",
      "commentUrn": "urn:li:comment:(urn:li:activity:123,6636062862760562688)",
      "actor": "urn:li:person:xxx",
      "message": "Great insights! Thanks for sharing.",
      "created": { "time": 1582160678569 },
      "likesSummary": { "totalLikes": 5 },
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

---

### linkedin_post_reactions

Retrieve reactions on a specific LinkedIn post.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123456` or `urn:li:ugcPost:123456`) |
| `platformId` | string | Yes | Platform connection ID |
| `start` | number | No | Starting index for pagination (default: 0) |
| `count` | number | No | Number of reactions to retrieve (default: 50, max: 100) |

**Example prompts:**

```text
"Who liked my LinkedIn post?"
"Show reactions on this post"
"Get all reactions on my announcement"
"List people who reacted to my post"
```

**Python example:**

```python
async def get_post_reactions():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("linkedin_post_reactions", {
                "postedId": "urn:li:ugcPost:7429953213384187904",
                "platformId": "linkedin-abc123",
                "count": 100
            })
            print(result.content[0].text)
```

**Response example:**

```json
{
  "success": true,
  "reactions": [
    {
      "id": "urn:li:reaction:(urn:li:person:xxx,urn:li:activity:123)",
      "reactionType": "LIKE",
      "actor": "urn:li:person:xxx",
      "created": { "time": 1686183251857 }
    },
    {
      "id": "urn:li:reaction:(urn:li:person:yyy,urn:li:activity:123)",
      "reactionType": "PRAISE",
      "actor": "urn:li:person:yyy",
      "created": { "time": 1686183300000 }
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

**Reaction types returned:**

| Type | Description |
|------|-------------|
| `LIKE` | Standard thumbs up |
| `PRAISE` | Clapping hands |
| `EMPATHY` | Heart/love |
| `INTEREST` | Lightbulb (insightful) |
| `APPRECIATION` | Thank you |
| `ENTERTAINMENT` | Funny/laughing |

---

## Workspace Tools (B2B)

### list_workspace_users

List team members in your workspace.

**Parameters:** None

**Example prompts:**

```text
"Show my team members"
"Who's in my workspace?"
"List workspace users"
```

**Python example:**

```python
async def list_workspace_users():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("list_workspace_users", {})
            print(result.content[0].text)
```

---

### create_workspace_user

Add a new user to your workspace.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `username` | string | Yes | Email address |
| `displayName` | string | No | Display name for the user |

**Example prompts:**

```text
"Add john@example.com to my workspace"
"Create a new team member"
"Invite marketing@company.com to join"
```

**Python example:**

```python
async def add_workspace_user():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.publora.com", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            result = await session.call_tool("create_workspace_user", {
                "username": "newuser@company.com",
                "displayName": "New User"
            })
            print(result.content[0].text)
```

---

### workspace_detach_user

Remove a user from your workspace.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | string | Yes | User ID to remove |

**Example prompts:**

```text
"Remove user xyz from my workspace"
"Detach this team member"
```

---

## Error Handling

When a tool encounters an error, it throws an exception that the MCP client will receive. Common error messages include:

| Error Message | Cause |
|---------------|-------|
| `"API key required..."` | Missing or invalid API key |
| `"Platform ID not found"` | Invalid platform connection ID |
| `"Post not found"` | Invalid post group ID |
| `"API error: 429"` | Rate limited by the API |
| `"API error: 502"` | Platform API temporarily unavailable |

**Example error handling in Python:**

```python
try:
    result = await session.call_tool("create_post", {
        "content": "Hello world!",
        "platforms": ["linkedin-abc123"],
        "scheduledTime": "2026-03-01T14:00:00Z"
    })
    print(result.content[0].text)
except Exception as e:
    print(f"Tool error: {e}")
```

---

## Best Practices

1. **Always get connections first** — Use `list_connections` to get valid platform IDs before creating posts

2. **Use ISO 8601 dates** — All dates should be in format `2026-03-01T14:00:00Z`

3. **Respect character limits** — Check platform limits before creating posts

4. **Handle errors gracefully** — Check for error responses in tool results

5. **Batch operations** — When posting to multiple platforms, include all platform IDs in a single `create_post` call
