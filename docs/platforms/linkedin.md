# LinkedIn API - Post to LinkedIn via REST API

Post to LinkedIn programmatically using the Publora REST API. A simpler alternative to the LinkedIn Marketing API, LinkedIn Share API, or LinkedIn Community Management API.

## LinkedIn API Overview

Publora provides a unified REST API for professional content publishing to LinkedIn, including text posts, media attachments (images and videos), analytics retrieval (impressions, reactions, comments), and reaction management. No need to manage LinkedIn OAuth flows, handle the LinkedIn API partner program requirements, or implement complex share creation endpoints.

### Why Use Publora Instead of LinkedIn Marketing API?

| Feature | Publora API | LinkedIn Marketing API |
|---------|-------------|------------------------|
| Authentication | Single API key | Complex OAuth 2.0 flow |
| API access | Instant | LinkedIn partner approval |
| Analytics | Built-in | Separate API calls |
| Multi-platform | Post to 10 platforms | LinkedIn only |
| Setup time | 5 minutes | Weeks (partner approval) |
| Reactions | Full support | Full support |

### Keywords: LinkedIn API, LinkedIn posting API, LinkedIn Share API, LinkedIn Marketing API, post to LinkedIn programmatically, LinkedIn REST API, LinkedIn developer API, LinkedIn automation API, LinkedIn content API, LinkedIn bot API, LinkedIn analytics API, LinkedIn UGC API

## Platform ID Format

```
linkedin-{profileId}
```

Where `{profileId}` is your LinkedIn profile identifier assigned during account connection via OAuth.

## Requirements

- A LinkedIn account connected via OAuth through the Publora dashboard
- API key from Publora

## Supported Content

| Type | Supported | Limits |
|------|-----------|--------|
| Text | Yes | 3,000 characters |
| Images | Yes | JPEG, PNG (WebP auto-converted to JPEG), multiple supported |
| Videos | Yes | MP4 format |
| Analytics | Yes | IMPRESSION, MEMBERS_REACHED, RESHARE, REACTION, COMMENT |
| Reactions | Yes | LIKE, PRAISE, EMPATHY, INTEREST, APPRECIATION, ENTERTAINMENT |
| Comments | Yes | Create, delete, reply (1,250 characters max) |
| Mentions | Yes | @mention people and organizations |

## Mentioning People and Organizations

Publora supports @mentioning LinkedIn members and organizations in your posts. When the post is published, the mention becomes a clickable link and the mentioned person/organization receives a notification.

### Mention Syntax

Use the following format in your post content:

```
@{urn:li:person:MEMBER_ID|Display Name}       # Mention a person
@{urn:li:organization:ORG_ID|Company Name}    # Mention an organization
```

### Examples

**Mentioning a person:**
```
Great insights from @{urn:li:person:4986615|Serge Bulaev} on building APIs!
```
Result: `Great insights from @Serge Bulaev on building APIs!`

**Mentioning a company:**
```
Excited to work with @{urn:li:organization:107107343|Creative Content Crafts Inc}!
```
Result: `Excited to work with @Creative Content Crafts Inc!`

Both mentions will be rendered as clickable links on LinkedIn.

### How to Find LinkedIn URNs

- **Person URN**: The numeric ID from LinkedIn's API. You can find this via LinkedIn's API or third-party tools. The format is `urn:li:person:{numeric_id}`.
- **Organization URN**: Found in the company page URL or via LinkedIn's API. Format: `urn:li:organization:{numeric_id}`.

> **Note:** Publora automatically converts the person URN format for LinkedIn's API compatibility.

### Important: Name Matching Requirements

**The display name must exactly match the name on LinkedIn** (case-sensitive):

- For **people**: Use their exact name as shown on their LinkedIn profile
- For **organizations**: Use the **exact registered company name** including suffixes like "Inc", "LLC", etc.

| Correct | Incorrect |
|---------|-----------|
| `@{urn:li:organization:123\|Acme Corp Inc}` | `@{urn:li:organization:123\|Acme Corp}` |
| `@{urn:li:person:456\|John Smith}` | `@{urn:li:person:456\|john smith}` |

If the name doesn't match exactly, the mention will appear as plain text instead of a clickable link.

### Code Example

**JavaScript**
```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Excited to collaborate with @{urn:li:person:4986615|Serge Bulaev} on this project!',
    platforms: ['linkedin-987654321']
  })
});
```

**Python**
```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': 'Excited to collaborate with @{urn:li:person:4986615|Serge Bulaev} on this project!',
        'platforms': ['linkedin-987654321']
    }
)
```

## Analytics

Publora can retrieve analytics for your LinkedIn posts. Available metrics:

| Metric | Description |
|--------|-------------|
| `IMPRESSION` | Number of times the post was displayed |
| `MEMBERS_REACHED` | Unique LinkedIn members who saw the post |
| `RESHARE` | Number of times the post was shared |
| `REACTION` | Total reactions on the post |
| `COMMENT` | Number of comments on the post |

## Reactions

LinkedIn supports a richer set of reactions than a simple "like":

| Reaction | Description |
|----------|-------------|
| `LIKE` | Standard like (thumbs up) |
| `PRAISE` | Clapping hands / applause |
| `EMPATHY` | Heart / love |
| `INTEREST` | Lightbulb / insightful |
| `APPRECIATION` | Supportive |
| `ENTERTAINMENT` | Funny / laughing |

### Create a Reaction

**Endpoint:** `POST /api/v1/linkedin-reactions`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123` or `urn:li:ugcPost:123`) |
| `reactionType` | string | Yes | One of: `LIKE`, `PRAISE`, `EMPATHY`, `INTEREST`, `APPRECIATION`, `ENTERTAINMENT` |
| `platformId` | string | Yes | Your LinkedIn platform ID (e.g., `linkedin-ABC123`) |

**JavaScript**
```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-reactions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:ugcPost:7429953213384187904',
    reactionType: 'INTEREST',
    platformId: 'linkedin-987654321'
  })
});

const data = await response.json();
console.log(data);
// {
//   success: true,
//   reaction: {
//     id: "urn:li:reaction:(urn:li:person:xxx,urn:li:ugcPost:xxx)",
//     reactionType: "INTEREST",
//     ...
//   }
// }
```

**cURL**
```bash
curl -X POST https://api.publora.com/api/v1/linkedin-reactions \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:ugcPost:7429953213384187904",
    "reactionType": "INTEREST",
    "platformId": "linkedin-987654321"
  }'
```

### Delete a Reaction

**Endpoint:** `DELETE /api/v1/linkedin-reactions`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN |
| `platformId` | string | Yes | Your LinkedIn platform ID |

**cURL**
```bash
curl -X DELETE https://api.publora.com/api/v1/linkedin-reactions \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:ugcPost:7429953213384187904",
    "platformId": "linkedin-987654321"
  }'
```

## Comments

Publora supports creating and deleting comments on LinkedIn posts programmatically.

### Create a Comment

**Endpoint:** `POST /api/v1/linkedin-comments`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN (e.g., `urn:li:share:123` or `urn:li:ugcPost:123`) |
| `message` | string | Yes | Comment text (max 1,250 characters) |
| `platformId` | string | Yes | Your LinkedIn platform ID (e.g., `linkedin-ABC123`) |
| `parentComment` | string | No | Comment URN for nested replies |

**JavaScript**
```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7434685316856377344',
    message: 'Great post! Thanks for sharing.',
    platformId: 'linkedin-987654321'
  })
});

const data = await response.json();
console.log(data);
// {
//   success: true,
//   comment: {
//     id: "7434695495614312448",
//     commentUrn: "urn:li:comment:(urn:li:activity:xxx,7434695495614312448)",
//     message: { text: "Great post! Thanks for sharing." },
//     ...
//   }
// }
```

**cURL**
```bash
curl -X POST https://api.publora.com/api/v1/linkedin-comments \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:share:7434685316856377344",
    "message": "Great post! Thanks for sharing.",
    "platformId": "linkedin-987654321"
  }'
```

### Delete a Comment

**Endpoint:** `DELETE /api/v1/linkedin-comments`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `postedId` | string | Yes | LinkedIn post URN the comment belongs to |
| `commentId` | string | Yes | Comment URN to delete |
| `platformId` | string | Yes | Your LinkedIn platform ID |

**JavaScript**
```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'DELETE',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7434685316856377344',
    commentId: 'urn:li:comment:(urn:li:activity:xxx,7434695495614312448)',
    platformId: 'linkedin-987654321'
  })
});

const data = await response.json();
console.log(data);
// { success: true, deleted: "urn:li:comment:(...)" }
```

**cURL**
```bash
curl -X DELETE https://api.publora.com/api/v1/linkedin-comments \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "postedId": "urn:li:share:7434685316856377344",
    "commentId": "urn:li:comment:(urn:li:activity:xxx,7434695495614312448)",
    "platformId": "linkedin-987654321"
  }'
```

### Replying to a Comment

To reply to an existing comment (nested comment), include the `parentComment` parameter:

```javascript
const response = await fetch('https://api.publora.com/api/v1/linkedin-comments', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    postedId: 'urn:li:share:7434685316856377344',
    message: 'I agree with this point!',
    platformId: 'linkedin-987654321',
    parentComment: 'urn:li:comment:(urn:li:activity:xxx,7434695495614312448)'
  })
});
```

## Examples

### Post a Text Update

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Excited to announce our Series A funding! We are building the future of social media management for developer teams. More details coming soon.',
    platforms: ['linkedin-987654321']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': 'Excited to announce our Series A funding! We are building the future of social media management for developer teams. More details coming soon.',
        'platforms': ['linkedin-987654321']
    }
)

data = response.json()
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "Excited to announce our Series A funding! We are building the future of social media management for developer teams. More details coming soon.",
    "platforms": ["linkedin-987654321"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Excited to announce our Series A funding! We are building the future of social media management for developer teams. More details coming soon.',
  platforms: ['linkedin-987654321']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

### Post with an Image

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Our team just wrapped up an incredible hackathon weekend. Here are some highlights from the event!',
    platforms: ['linkedin-987654321']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': 'Our team just wrapped up an incredible hackathon weekend. Here are some highlights from the event!',
        'platforms': ['linkedin-987654321']
    }
)

data = response.json()
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "Our team just wrapped up an incredible hackathon weekend. Here are some highlights from the event!",
    "platforms": ["linkedin-987654321"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Our team just wrapped up an incredible hackathon weekend. Here are some highlights from the event!',
  platforms: ['linkedin-987654321']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

> **Note:** To attach media to a LinkedIn post, first create the post, then upload media using the [media upload workflow](../guides/media-uploads.md) with the returned `postGroupId`.

### Check Post Analytics

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/post-analytics?postId=post-abc123&platform=linkedin-987654321', {
  method: 'GET',
  headers: {
    'x-publora-key': 'YOUR_API_KEY'
  }
});

const analytics = await response.json();
console.log(analytics);
// {
//   impressions: 4521,
//   membersReached: 3200,
//   reshares: 12,
//   reactions: 89,
//   comments: 15
// }
```

**Python (requests)**

```python
import requests

response = requests.get(
    'https://api.publora.com/api/v1/post-analytics',
    headers={
        'x-publora-key': 'YOUR_API_KEY'
    },
    params={
        'postId': 'post-abc123',
        'platform': 'linkedin-987654321'
    }
)

analytics = response.json()
print(analytics)
```

**cURL**

```bash
curl -G https://api.publora.com/api/v1/post-analytics \
  -H "x-publora-key: YOUR_API_KEY" \
  -d "postId=post-abc123" \
  -d "platform=linkedin-987654321"
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.get('https://api.publora.com/api/v1/post-analytics', {
  headers: {
    'x-publora-key': 'YOUR_API_KEY'
  },
  params: {
    postId: 'post-abc123',
    platform: 'linkedin-987654321'
  }
});

console.log(response.data);
```

## Platform Quirks

- **URN formats differ**: LinkedIn URLs use `urn:li:activity:xxx` but the API requires `urn:li:share:xxx` or `urn:li:ugcPost:xxx`. For posts created via Publora, use the `postedId` from the `get-post` endpoint. For external posts, you may need to look up the correct URN format.
- **WebP auto-conversion**: If you provide a WebP image URL, Publora automatically converts it to JPEG before uploading to LinkedIn. No action needed on your part.
- **3,000-character limit**: LinkedIn enforces a strict 3,000-character limit for post text. Publora will return an error if your content exceeds this.
- **Multiple images**: LinkedIn supports posting multiple images at once. They will appear as a carousel or multi-image post depending on the count.
- **Rich text not supported via API**: LinkedIn's API does not support bold, italic, or other rich text formatting. Use plain text or Unicode characters for emphasis.
- **Analytics delay**: LinkedIn analytics may take up to 24 hours to fully populate. Querying immediately after posting will return partial data.
- **Hashtags**: LinkedIn hashtags are supported in the content body. They are treated as plain text but become clickable on the platform.
- **Reactions on external posts**: You can only react to posts visible in your LinkedIn network. Ensure your account is connected to the post author.

## Character Limits

| Element | Limit |
|---------|-------|
| Post body | 3,000 characters |
| Comment | 1,250 characters |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
