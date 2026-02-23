# Upload Media

Upload images and videos to attach to posts. Uses pre-signed S3 URLs for direct uploads.

## Endpoint

```
POST https://api.publora.com/api/v1/get-upload-url
```

## Headers

| Header | Required | Description |
|--------|----------|-------------|
| `x-publora-key` | Yes | Your API key |
| `x-publora-user-id` | No | Managed user ID (workspace only) |
| `Content-Type` | Yes | `application/json` |

## Request Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fileName` | string | Yes | Name of the file (e.g., `photo.jpg`) |
| `contentType` | string | Yes | MIME type (e.g., `image/jpeg`, `video/mp4`) |
| `type` | string | Yes | `"image"` or `"video"` |
| `postGroupId` | string | Yes | The post group to attach this media to |

## Response

```json
{
  "success": true,
  "uploadUrl": "https://brandcraft-media.s3.amazonaws.com/images/...",
  "fileUrl": "https://media.publora.com/images/...",
  "mediaId": "507f1f77bcf86cd799439099"
}
```

| Field | Description |
|-------|-------------|
| `uploadUrl` | Pre-signed S3 URL. Upload your file here via HTTP PUT. Expires in 1 hour. |
| `fileUrl` | Public URL of the file after upload. |
| `mediaId` | Media record ID for tracking. |

## Upload Flow

> **Important:** When uploading media, always create the post as a **draft first**, upload media, then schedule. This prevents the scheduler from processing the post before media upload completes.

### Recommended workflow (with media):

```
1. POST /create-post              → Create draft (no scheduledTime), get postGroupId
2. POST /get-upload-url           → Get pre-signed URL
3. PUT {uploadUrl}                → Upload file to S3
4. PUT /update-post/:postGroupId  → Set status="scheduled" and scheduledTime
```

### Why this matters

If you create a post with `scheduledTime` set immediately, the scheduler may attempt to publish before your media upload completes — resulting in a failed post or missing media.

### Quick workflow (text-only posts):

```
1. POST /create-post              → Create with scheduledTime (no media needed)
```

## Supported Formats

| Type | Formats | Max Size |
|------|---------|----------|
| Image | JPEG, PNG, GIF, WebP | 512 MB per file |
| Video | MP4, MOV | 512 MB per file |

**Limits:** Up to 4 images OR 1 video per post.

**Note:** WebP images are automatically converted to JPEG for platforms that don't support WebP (LinkedIn, Telegram, Bluesky).

## Examples

### Complete workflow: Post with image

#### JavaScript (fetch)

```javascript
const API_KEY = 'YOUR_API_KEY';
const BASE_URL = 'https://api.publora.com/api/v1';

// Step 1: Create draft post (no scheduledTime)
const postRes = await fetch(`${BASE_URL}/create-post`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-publora-key': API_KEY },
  body: JSON.stringify({
    content: 'Check out our new product! 🚀',
    platforms: ['twitter-123456789', 'linkedin-ABC123']
    // No scheduledTime = draft
  })
});
const { postGroupId } = await postRes.json();

// Step 2: Get upload URL
const urlRes = await fetch(`${BASE_URL}/get-upload-url`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-publora-key': API_KEY },
  body: JSON.stringify({
    fileName: 'product-photo.jpg',
    contentType: 'image/jpeg',
    type: 'image',
    postGroupId
  })
});
const { uploadUrl, fileUrl } = await urlRes.json();

// Step 3: Upload file to S3
const fileBuffer = await fs.promises.readFile('./product-photo.jpg');
await fetch(uploadUrl, {
  method: 'PUT',
  headers: { 'Content-Type': 'image/jpeg' },
  body: fileBuffer
});
console.log(`Uploaded: ${fileUrl}`);

// Step 4: Schedule the post
await fetch(`${BASE_URL}/update-post/${postGroupId}`, {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json', 'x-publora-key': API_KEY },
  body: JSON.stringify({
    status: 'scheduled',
    scheduledTime: '2026-03-01T14:00:00.000Z'
  })
});
console.log('Post scheduled!');
```

#### Python (requests)

```python
import requests

API_KEY = 'YOUR_API_KEY'
BASE_URL = 'https://api.publora.com/api/v1'
headers = {'Content-Type': 'application/json', 'x-publora-key': API_KEY}

# Step 1: Create draft post
post_res = requests.post(f'{BASE_URL}/create-post', headers=headers, json={
    'content': 'Check out our new product! 🚀',
    'platforms': ['twitter-123456789', 'linkedin-ABC123']
})
post_group_id = post_res.json()['postGroupId']

# Step 2: Get upload URL
url_res = requests.post(f'{BASE_URL}/get-upload-url', headers=headers, json={
    'fileName': 'product-photo.jpg',
    'contentType': 'image/jpeg',
    'type': 'image',
    'postGroupId': post_group_id
})
data = url_res.json()

# Step 3: Upload file to S3
with open('product-photo.jpg', 'rb') as f:
    requests.put(data['uploadUrl'], headers={'Content-Type': 'image/jpeg'}, data=f)
print(f"Uploaded: {data['fileUrl']}")

# Step 4: Schedule the post
requests.put(f'{BASE_URL}/update-post/{post_group_id}', headers=headers, json={
    'status': 'scheduled',
    'scheduledTime': '2026-03-01T14:00:00.000Z'
})
print('Post scheduled!')
```

#### cURL

```bash
API_KEY="YOUR_API_KEY"

# Step 1: Create draft post
POST_GROUP_ID=$(curl -s -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Check out our new product! 🚀",
    "platforms": ["twitter-123456789", "linkedin-ABC123"]
  }' | jq -r '.postGroupId')

# Step 2: Get upload URL
UPLOAD_URL=$(curl -s -X POST https://api.publora.com/api/v1/get-upload-url \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d "{
    \"fileName\": \"product-photo.jpg\",
    \"contentType\": \"image/jpeg\",
    \"type\": \"image\",
    \"postGroupId\": \"$POST_GROUP_ID\"
  }" | jq -r '.uploadUrl')

# Step 3: Upload file to S3
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @product-photo.jpg

# Step 4: Schedule the post
curl -X PUT "https://api.publora.com/api/v1/update-post/$POST_GROUP_ID" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "status": "scheduled",
    "scheduledTime": "2026-03-01T14:00:00.000Z"
  }'
```

### Upload a video

#### JavaScript (fetch)

```javascript
// Step 1: Get upload URL for video
const urlResponse = await fetch('https://api.publora.com/api/v1/get-upload-url', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    fileName: 'promo-video.mp4',
    contentType: 'video/mp4',
    type: 'video',
    postGroupId: '507f1f77bcf86cd799439011'
  })
});
const { uploadUrl } = await urlResponse.json();

// Step 2: Upload video to S3
const videoBuffer = await fs.promises.readFile('./promo-video.mp4');
await fetch(uploadUrl, {
  method: 'PUT',
  headers: { 'Content-Type': 'video/mp4' },
  body: videoBuffer
});
```

#### Python (requests)

```python
# Step 1: Get upload URL for video
url_response = requests.post(
    'https://api.publora.com/api/v1/get-upload-url',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'fileName': 'promo-video.mp4',
        'contentType': 'video/mp4',
        'type': 'video',
        'postGroupId': '507f1f77bcf86cd799439011'
    }
)
upload_url = url_response.json()['uploadUrl']

# Step 2: Upload video to S3
with open('promo-video.mp4', 'rb') as f:
    requests.put(upload_url, headers={'Content-Type': 'video/mp4'}, data=f)
```

### Upload multiple images for a carousel

```javascript
const images = ['photo1.jpg', 'photo2.jpg', 'photo3.jpg', 'photo4.jpg'];

for (const image of images) {
  // Get upload URL for each image
  const urlRes = await fetch('https://api.publora.com/api/v1/get-upload-url', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    },
    body: JSON.stringify({
      fileName: image,
      contentType: 'image/jpeg',
      type: 'image',
      postGroupId: '507f1f77bcf86cd799439011'
    })
  });
  const { uploadUrl } = await urlRes.json();

  // Upload each image
  const fileBuffer = await fs.promises.readFile(`./${image}`);
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': 'image/jpeg' },
    body: fileBuffer
  });
}
// All 4 images now attached to the post group
```

## Errors

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"fileName is required"` | Missing fileName |
| 400 | `"contentType is required"` | Missing contentType |
| 400 | `"type is required"` | Missing type parameter |
| 400 | `"postGroupId is required"` | Missing postGroupId |
| 400 | `"Invalid content type"` | Not an image/* or video/* MIME type |
| 400 | `"Invalid type"` | type must be "image" or "video" |
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 500 | `"Server error"` | Internal server error during URL generation |

## Platform Media Limits

| Platform | Images | Videos | Notes |
|----------|--------|--------|-------|
| X / Twitter | Up to 4 | 1 per post | PNG preferred for images |
| LinkedIn | Multiple | 1 per post | WebP auto-converted to JPEG |
| Instagram | Carousel (10) | Reels or Stories | Business account required |
| Threads | Carousel | 1 per post | WebP auto-converted |
| TikTok | -- | 1 per post | MP4 only, min 23 FPS |
| YouTube | -- | 1 per post | MP4, streaming upload |
| Facebook | Multiple | 1 per post | Carousel support |
| Bluesky | Up to 4 | 1 per post | WebP auto-converted, alt text |
| Mastodon | Up to 4 | 1 per post | Standard limits |
| Telegram | Multiple | 1 per post | 1024 char caption max (bot) |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
