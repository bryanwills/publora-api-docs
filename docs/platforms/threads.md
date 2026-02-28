# Threads API - Post to Threads via REST API

Post to Threads (by Meta) programmatically using the Publora REST API. A simpler alternative to the official Threads API or Meta Graph API for Threads.

## Threads API Overview

Publora provides a unified REST API for publishing text posts, images, videos, carousels, and automatic thread splitting for long-form content on Threads. No need to manage Meta OAuth flows, handle the Threads Publishing API complexity, or wait for Threads API access approval.

### Why Use Publora Instead of Threads API / Meta Graph API?

| Feature | Publora API | Threads API (Meta) |
|---------|-------------|-------------------|
| Authentication | Single API key | Meta OAuth 2.0 flow |
| API access | Instant | Requires Meta app review |
| Thread creation | Automatic splitting | Manual implementation |
| Multi-platform | Post to 10 platforms | Threads only |
| Setup time | 5 minutes | Days to weeks |
| Carousel support | Yes | Yes |

### Keywords: Threads API, Threads posting API, Meta Threads API, post to Threads programmatically, Threads REST API, Threads developer API, Threads automation API, Threads bot API, Instagram Threads API, publish to Threads API

## Platform ID Format

```
threads-{accountId}
```

Where `{accountId}` is your Threads account ID assigned during connection via Meta OAuth.

## Requirements

- A Threads account connected via Meta OAuth through the Publora dashboard
- API key from Publora

## Supported Content

| Type | Supported | Limits |
|------|-----------|--------|
| Text | Yes | 500 characters |
| Images | Yes | Up to 20 per carousel, WebP auto-converted |
| Videos | Yes | MP4 format, up to 20 per carousel |
| Carousels | Yes | 2-20 images, videos, or mixed media |
| Threads | Yes | Auto-split for long content |
| Hashtags | Yes | Maximum 1 hashtag per post |

## Threading

When your content exceeds the 500-character limit, Publora automatically splits it into a thread (multiple connected posts):

### How It Works

Publora uses the official Threads API `reply_to_id` parameter to chain posts together. Each subsequent post is posted as a reply to the previous one, creating a connected thread visible on Threads.

**Technical flow:**
1. First post is published normally
2. Each subsequent post is published with `reply_to_id` set to the previous post's ID
3. All posts appear as a connected thread on Threads

### Automatic Splitting

When content exceeds 500 characters, Publora automatically splits it:

- Content is split at paragraph breaks (`\n\n`) when possible
- Falls back to sentence boundaries (`. `, `! `, `? `)
- Falls back to word boundaries if needed
- Each part respects the 500-character limit

### Manual Thread Parts

You can manually define where thread breaks should occur using either method:

**Method 1: Triple dash separator**
```
This is my first post in the thread.

---

This is my second post in the thread.

---

And this is my third post!
```

**Method 2: Explicit markers**
```
First part of the thread [1/3]

Second part of the thread [2/3]

Third and final part [3/3]
```

When explicit `[n/m]` markers are detected, Publora preserves them exactly as written and splits at those points.

### Media in Threads

- **Carousel/Images:** Attached to the first post only
- **Video:** Attached to the first post only
- Subsequent posts in the thread are text-only

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
    content: 'Just shipped a major update to our API. Faster response times, better error messages, and new endpoints for batch operations.',
    platforms: ['threads-55667788']
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
        'content': 'Just shipped a major update to our API. Faster response times, better error messages, and new endpoints for batch operations.',
        'platforms': ['threads-55667788']
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
    "content": "Just shipped a major update to our API. Faster response times, better error messages, and new endpoints for batch operations.",
    "platforms": ["threads-55667788"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Just shipped a major update to our API. Faster response times, better error messages, and new endpoints for batch operations.',
  platforms: ['threads-55667788']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

### Post with an Image Carousel

Carousels require 2-20 images or videos. The workflow is: create a draft post, upload images, then schedule.

**JavaScript (fetch)**

```javascript
const API_KEY = 'YOUR_API_KEY';
const BASE_URL = 'https://api.publora.com/api/v1';

// Step 1: Create a draft post (no scheduledTime)
const postResponse = await fetch(`${BASE_URL}/create-post`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    content: 'Our product evolution over the past year. #buildinpublic',
    platforms: ['threads-55667788']
    // No scheduledTime = draft
  })
});

const { postGroupId } = await postResponse.json();

// Step 2: Upload each image (2-20 images supported)
const images = ['photo1.jpg', 'photo2.jpg', 'photo3.jpg'];

for (const fileName of images) {
  // Get upload URL
  const uploadUrlResponse = await fetch(`${BASE_URL}/get-upload-url`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': API_KEY
    },
    body: JSON.stringify({
      fileName,
      contentType: 'image/jpeg',
      type: 'image',
      postGroupId
    })
  });

  const { uploadUrl } = await uploadUrlResponse.json();

  // Upload to S3
  const fileBuffer = await fs.promises.readFile(`./${fileName}`);
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': 'image/jpeg' },
    body: fileBuffer
  });
}

// Step 3: Schedule the post
await fetch(`${BASE_URL}/update-post/${postGroupId}`, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    status: 'scheduled',
    scheduledTime: '2026-03-15T14:00:00.000Z'
  })
});

console.log('Carousel scheduled!');
```

**Python (requests)**

```python
import requests

API_KEY = 'YOUR_API_KEY'
BASE_URL = 'https://api.publora.com/api/v1'
HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
}

# Step 1: Create a draft post (no scheduledTime)
post_response = requests.post(
    f'{BASE_URL}/create-post',
    headers=HEADERS,
    json={
        'content': 'Our product evolution over the past year. #buildinpublic',
        'platforms': ['threads-55667788']
        # No scheduledTime = draft
    }
)

post_group_id = post_response.json()['postGroupId']

# Step 2: Upload each image (2-20 images supported)
images = ['photo1.jpg', 'photo2.jpg', 'photo3.jpg']

for file_name in images:
    # Get upload URL
    upload_response = requests.post(
        f'{BASE_URL}/get-upload-url',
        headers=HEADERS,
        json={
            'fileName': file_name,
            'contentType': 'image/jpeg',
            'type': 'image',
            'postGroupId': post_group_id
        }
    )

    upload_url = upload_response.json()['uploadUrl']

    # Upload to S3
    with open(f'./{file_name}', 'rb') as f:
        requests.put(upload_url, headers={'Content-Type': 'image/jpeg'}, data=f.read())

# Step 3: Schedule the post
requests.put(
    f'{BASE_URL}/update-post/{post_group_id}',
    headers=HEADERS,
    json={
        'status': 'scheduled',
        'scheduledTime': '2026-03-15T14:00:00.000Z'
    }
)

print('Carousel scheduled!')
```

**cURL**

```bash
API_KEY="YOUR_API_KEY"

# Step 1: Create a draft post (no scheduledTime)
POST_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Our product evolution over the past year. #buildinpublic",
    "platforms": ["threads-55667788"]
  }')

POST_GROUP_ID=$(echo "$POST_RESPONSE" | jq -r '.postGroupId')

# Step 2: Upload each image (2-20 images supported)
for FILE in photo1.jpg photo2.jpg photo3.jpg; do
  UPLOAD_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/get-upload-url \
    -H "Content-Type: application/json" \
    -H "x-publora-key: $API_KEY" \
    -d "{
      \"fileName\": \"$FILE\",
      \"contentType\": \"image/jpeg\",
      \"type\": \"image\",
      \"postGroupId\": \"$POST_GROUP_ID\"
    }")

  UPLOAD_URL=$(echo "$UPLOAD_RESPONSE" | jq -r '.uploadUrl')

  curl -s -X PUT "$UPLOAD_URL" \
    -H "Content-Type: image/jpeg" \
    --data-binary @"./$FILE"
done

# Step 3: Schedule the post
curl -X PUT "https://api.publora.com/api/v1/update-post/$POST_GROUP_ID" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "status": "scheduled",
    "scheduledTime": "2026-03-15T14:00:00.000Z"
  }'
```

> **Note:** For immediate publishing, set `scheduledTime` to the current time or a few seconds in the future. For more details, see the [media upload workflow](../guides/media-uploads.md).

### Post a Thread (Long Content)

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: `We just completed a major infrastructure migration and I want to share what we learned. Moving from a monolithic architecture to microservices is not as straightforward as the blog posts make it sound.

First, we had to map every single dependency between our services. This alone took two weeks. We discovered circular dependencies we never knew existed and had to refactor several core modules before we could even begin the migration.

The actual migration took three months. We ran both systems in parallel, comparing outputs in real-time. When we finally cut over, we had 99.97% uptime throughout the process. The key was incremental rollout and comprehensive monitoring at every step.`,
    platforms: ['threads-55667788']
  })
});

const data = await response.json();
console.log(data);
```

**Python (requests)**

```python
import requests

content = """We just completed a major infrastructure migration and I want to share what we learned. Moving from a monolithic architecture to microservices is not as straightforward as the blog posts make it sound.

First, we had to map every single dependency between our services. This alone took two weeks. We discovered circular dependencies we never knew existed and had to refactor several core modules before we could even begin the migration.

The actual migration took three months. We ran both systems in parallel, comparing outputs in real-time. When we finally cut over, we had 99.97% uptime throughout the process. The key was incremental rollout and comprehensive monitoring at every step."""

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'content': content,
        'platforms': ['threads-55667788']
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
    "content": "We just completed a major infrastructure migration and I want to share what we learned. Moving from a monolithic architecture to microservices is not as straightforward as the blog posts make it sound.\n\nFirst, we had to map every single dependency between our services. This alone took two weeks. We discovered circular dependencies we never knew existed and had to refactor several core modules before we could even begin the migration.\n\nThe actual migration took three months. We ran both systems in parallel, comparing outputs in real-time. When we finally cut over, we had 99.97% uptime throughout the process. The key was incremental rollout and comprehensive monitoring at every step.",
    "platforms": ["threads-55667788"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const content = `We just completed a major infrastructure migration and I want to share what we learned. Moving from a monolithic architecture to microservices is not as straightforward as the blog posts make it sound.

First, we had to map every single dependency between our services. This alone took two weeks. We discovered circular dependencies we never knew existed and had to refactor several core modules before we could even begin the migration.

The actual migration took three months. We ran both systems in parallel, comparing outputs in real-time. When we finally cut over, we had 99.97% uptime throughout the process. The key was incremental rollout and comprehensive monitoring at every step.`;

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content,
  platforms: ['threads-55667788']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

Publora will automatically split this into multiple thread posts, each staying within the 500-character limit.

## Platform Quirks

- **Single hashtag limit**: Threads allows a maximum of 1 hashtag per post. If your content includes more than one hashtag, only the first will be recognized by the platform.
- **WebP auto-conversion**: If you provide WebP images, Publora automatically converts them before uploading to Threads.
- **Auto-threading**: Content exceeding 500 characters is automatically split into a thread. Publora splits at sentence boundaries to keep posts readable.
- **Manual thread parts**: You can use `---` as a separator in your content to explicitly define where thread breaks should occur.
- **No edit support**: Once posted, Threads posts cannot be edited via the API. You would need to delete and repost.
- **MP4 for videos**: Only MP4 video format is supported. Other formats will be rejected.

## Character Limits

| Element | Limit |
|---------|-------|
| Post body | 500 characters |
| Hashtags | 1 per post |
| Carousel items | 2-20 images or videos |
| Thread parts | No fixed limit on number of parts |

## Rate Limits

The underlying Threads API has the following publishing limits:

| Limit Type | Value |
|------------|-------|
| Posts per 24 hours | 250 |
| Posts per hour | ~25 (varies) |

Publora handles rate limiting automatically and will return appropriate errors if limits are exceeded.


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
