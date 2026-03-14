# How to Post Comments on LinkedIn with Publora

Post comments on LinkedIn posts programmatically using the Publora API. Engage with your network automatically without manual interaction.

## Overview

Publora allows you to create and delete comments on any LinkedIn post visible to your connected account. You can also reply to existing comments (nested replies).

## Prerequisites

- Publora API key
- LinkedIn account connected via Publora dashboard
- The `postedId` (LinkedIn URN) of the post you want to comment on

## Create a Comment

**Endpoint:** `POST https://api.publora.com/api/v1/linkedin-comments`

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123` or `urn:li:ugcPost:123`) |
| `message` | string | Yes | Comment text (max 1,250 characters) |
| `platformId` | string | Yes | Your LinkedIn platform ID (e.g., `linkedin-ABC123`) |
| `parentComment` | string | No | Comment URN for nested replies |

### JavaScript Example

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:ugcPost:7429953213384187904',
    message: 'Great insights! Thanks for sharing.',
    platformId: 'linkedin-ABC123'
  })
});

const data = await response.json();
console.log(data);
// {
//   success: true,
//   comment: {
//     id: "7434695495614312448",
//     commentUrn: "urn:li:comment:(urn:li:ugcPost:xxx,7434695495614312448)",
//     message: "Great insights! Thanks for sharing."
//   }
// }
```

### Python Example

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/linkedin-comments',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'postedId': 'urn:li:ugcPost:7429953213384187904',
        'message': 'Great insights! Thanks for sharing.',
        'platformId': 'linkedin-ABC123'
    }
)

print(response.json())
```

### cURL Example

```bash
curl -X POST https://api.publora.com/api/v1/linkedin-comments \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:ugcPost:7429953213384187904",
    "message": "Great insights! Thanks for sharing.",
    "platformId": "linkedin-ABC123"
  }'
```

## Reply to a Comment

To reply to an existing comment, include the `parentComment` parameter:

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:ugcPost:7429953213384187904',
    message: 'I completely agree with your point!',
    platformId: 'linkedin-ABC123',
    parentComment: 'urn:li:comment:(urn:li:ugcPost:xxx,7434695495614312448)'
  })
});
```

## Delete a Comment

**Endpoint:** `DELETE https://api.publora.com/api/v1/linkedin-comments`

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN the comment belongs to |
| `commentId` | string | Yes | Comment URN to delete |
| `platformId` | string | Yes | Your LinkedIn platform ID |

### JavaScript Example

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'DELETE',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:ugcPost:7429953213384187904',
    commentId: 'urn:li:comment:(urn:li:ugcPost:xxx,7434695495614312448)',
    platformId: 'linkedin-ABC123'
  })
});

const data = await response.json();
// { success: true, deleted: "urn:li:comment:(...)" }
```

## Getting the postedId

For posts created via Publora, get the `postedId` from the get-post endpoint:

```javascript
const response = await fetch('https://api.publora.com/api/v1/get-post/YOUR_POST_GROUP_ID', {
  headers: { 'x-publora-key': 'YOUR_API_KEY' }
});

const data = await response.json();
const postedId = data.posts[0].postedId; // e.g., "urn:li:share:7434685316856377344"
```

## Important Notes

- **Character limit:** Comments are limited to 1,250 characters
- **Network visibility:** You can only comment on posts visible to your LinkedIn account
- **URN format:** Use `urn:li:share:xxx` or `urn:li:ugcPost:xxx` format (not `urn:li:activity:xxx` from URLs)

## Related Guides

- [LinkedIn Reactions](/docs/guides/linkedin-reactions.md) - Like and react to posts
- [LinkedIn Mentions](/docs/guides/linkedin-mentions.md) - Mention users and companies
- [LinkedIn Platform Guide](/docs/platforms/linkedin.md) - Complete LinkedIn API reference
