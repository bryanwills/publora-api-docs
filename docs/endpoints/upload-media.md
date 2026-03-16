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
| `x-publora-client` | No | Set to `mcp` for MCP tool access |
| `Content-Type` | Yes | `application/json` |

> **MCP access:** If the `x-publora-client: mcp` header is sent, the server validates `entitlements.features.mcpAccess` and returns `403 "MCP access is not enabled for this account"` if the feature is not enabled.

## Request Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fileName` | string | Yes | Name of the file (e.g., `photo.jpg`). The filename is sanitized before use: whitespace is trimmed, spaces are replaced with underscores, and special characters (except underscores, dots, and hyphens) are removed. For example, `my photo (1).jpg` becomes `my_photo_1.jpg`. |
| `contentType` | string | Yes | MIME type (e.g., `image/jpeg`, `video/mp4`). The API route does **not** validate this parameter — any MIME type string is accepted. |
| `postGroupId` | string | Yes | The post group to attach this media to. **Security note:** The API does not verify that the `postGroupId` belongs to the requesting user. This is a known limitation. |
| `type` | string | No | Media type: `"image"` or `"video"`. Determines the S3 key prefix. **Warning:** Omitting `type` will result in an `undefined` S3 key — the server has no default, so neither the `"image"` nor `"video"` branch executes when `type` is not provided. Always include this parameter. |

## Response

```json
{
  "success": true,
  "uploadUrl": "https://brandcraft-media.s3.amazonaws.com/images/...",
  "fileUrl": "https://brandcraft-media.s3.amazonaws.com/images/1710500000000-product-photo.jpg",
  "mediaId": "65f8a1b2c3d4e5f6a7b8c9d0"
}
```

| Field | Description |
|-------|-------------|
| `success` | `true` if the upload URL was generated successfully. |
| `uploadUrl` | Pre-signed S3 URL. Upload your file here via HTTP PUT. Expires in 1 hour. |
| `fileUrl` | The public URL where the file will be accessible after upload. |
| `mediaId` | The unique ID of the created media record. Use this to reference or delete the media. |

> **Internal fields:** Media records also store additional internal fields (`urlPure`, `mimeType`, `addedAt`, `filePath`, `uploadUrl`, `fileName` (S3 key), `metadata`) that are not returned by the API. The `metadata` object contains: `size`, `width`, `height`, `aspectRatio`, `format`, `frameRate`, `codecName`, `bitRate`, and `duration`.

### Dashboard vs API Differences

The dashboard uses a different endpoint (`/media/generate-upload-url`) with different behavior:

| Aspect | API (`/api/v1/get-upload-url`) | Dashboard (`/media/generate-upload-url`) |
|--------|-------------------------------|------------------------------------------|
| **Required fields** | `fileName`, `contentType`, `postGroupId` | `fileName`, `contentType` (no `postGroupId` required) |
| **Validation** | No `contentType` validation | Validates `contentType` starts with `image/` or `video/` |
| **Error message** | `"fileName, contentType, and postGroupId are required"` | `"fileName and contentType are required"` |
| **Response format** | `{ success, uploadUrl, fileUrl, mediaId }` | `{ uploadUrl, key }` (no `success`, `fileUrl`, or `mediaId`) |
| **Auth** | API key (`x-publora-key`) | Session cookie |

## Upload Flow

> **Important:** When uploading media, always create the post as a **draft first**, upload media, then schedule. This prevents the scheduler from processing the post before media upload completes.

### Recommended workflow (with media):

```
1. POST /create-post                → Create draft (no scheduledTime), get postGroupId
2. POST /get-upload-url        → Get pre-signed URL
3. PUT {uploadUrl}                   → Upload file to S3
4. PUT /update-post/:postGroupId     → Set status="scheduled" and scheduledTime
```

### Why this matters

If you create a post with `scheduledTime` set immediately, the scheduler may attempt to publish before your media upload completes — resulting in a failed post or missing media.

### Quick workflow (text-only posts):

```
1. POST /create-post              → Create with scheduledTime (no media needed)
```

## Supported Formats

> **Note:** The formats listed below are what the publishing platforms accept, not what the upload endpoint enforces. The `get-upload-url` API route accepts any `contentType` value without validation.

| Type | Formats | Max Size |
|------|---------|----------|
| Image | JPEG, PNG, GIF, WebP | No enforced limit (pre-signed URL) |
| Video | MP4, MOV | No enforced limit (pre-signed URL) |

**Note:** The 512 MB per file limit applies only to the server-side multipart upload endpoint (`process-video`), not to pre-signed URL uploads. Individual platforms may impose their own size limits at publish time.

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
const { uploadUrl, fileUrl, mediaId } = await urlRes.json();

// Step 3: Upload file to S3
const fileBuffer = await fs.promises.readFile('./product-photo.jpg');
await fetch(uploadUrl, {
  method: 'PUT',
  headers: { 'Content-Type': 'image/jpeg' },
  body: fileBuffer
});
console.log(`Uploaded: ${fileUrl} (mediaId: ${mediaId})`);

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
print(f"Uploaded: {data['fileUrl']} (mediaId: {data['mediaId']})")

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
UPLOAD_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/get-upload-url \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d "{
    \"fileName\": \"product-photo.jpg\",
    \"contentType\": \"image/jpeg\",
    \"type\": \"image\",
    \"postGroupId\": \"$POST_GROUP_ID\"
  }")
UPLOAD_URL=$(echo "$UPLOAD_RESPONSE" | jq -r '.uploadUrl')
FILE_URL=$(echo "$UPLOAD_RESPONSE" | jq -r '.fileUrl')
MEDIA_ID=$(echo "$UPLOAD_RESPONSE" | jq -r '.mediaId')

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
const { uploadUrl, fileUrl, mediaId } = await urlResponse.json();

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
| 400 | `"fileName, contentType, and postGroupId are required"` | Missing `fileName`, `contentType`, or `postGroupId` |
| 400 | `"Invalid x-publora-user-id"` | The `x-publora-user-id` header value is not a valid ID |
| 401 | `"API key is required"` | `x-publora-key` header is missing entirely |
| 401 | `"Invalid API key"` | `x-publora-key` is present but invalid |
| 401 | `"Invalid API key owner"` | API key exists but the associated user account was not found |
| 403 | `"API access is not enabled for this account"` | The user's account does not have API access enabled |
| 403 | `"MCP access is not enabled for this account"` | The `x-publora-client: mcp` header was sent but `entitlements.features.mcpAccess` is not enabled |
| 403 | `"Workspace access is not enabled for this key"` | API key does not have workspace access enabled |
| 403 | `"User is not managed by key"` | Managed user does not belong to the API key owner's workspace |
| 500 | `"Failed to create upload URL"` | Internal server error during URL generation |

## Server-Side Video Upload

For video files, you can use the server-side upload endpoint which handles the upload and extracts video metadata (resolution, codec, frame rate, bitrate, duration).

> **Dashboard-only endpoint.** This endpoint uses session authentication (cookies) and is not available via API key auth. It is accessible only from the Publora dashboard.

### Endpoint

```
POST https://api.publora.com/media/process-video
```

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Cookie` | Yes | Active dashboard session cookie |
| `Content-Type` | Yes | `multipart/form-data` |

### Request Body (multipart/form-data)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `video` | file | Yes | The video file to upload |
| `postGroupId` | string | Yes | The post group to attach this video to |

This endpoint accepts `multipart/form-data` with a single video file attached (field name: `video`) and a `postGroupId`. The server uploads the file to S3 and extracts metadata automatically. Processing is asynchronous -- the endpoint returns a `sessionId` which can be used to track progress via SSE (Server-Sent Events).

> **Known issue:** The `process-video` endpoint does not pass the `type` parameter to the presigned URL generator, which results in an `undefined` S3 key — the key variable remains undefined when the type is neither `"image"` nor `"video"`, causing an S3 error. This is a backend bug being tracked for a fix.
>
> **Note:** The multer `fileFilter` rejection (for non-MOV/MP4 files) throws an error inside multer's callback. The resulting HTTP response format depends on Express's global error handler and may not produce a clean JSON 400 response.

### Response

```json
{
  "sessionId": "1710500000000"
}
```

The `sessionId` is returned for future use, but the SSE progress endpoint (`/processing-progress/:id`) is currently disabled (commented out in source). There is no working SSE endpoint to subscribe to yet.

**Limits:** 1 video file per request, 512 MB max. Only MP4 and MOV formats are accepted.

### Errors

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"No video file uploaded"` | No file is attached to the request |
| 400 | `"Only MOV/MP4 files are allowed for this endpoint"` | The uploaded file format is not MOV or MP4 |

## Delete Media

Remove a media file attached to a post.

> **Dashboard-only endpoint.** This endpoint uses session authentication (cookies) and is not available via API key auth. It is accessible only from the Publora dashboard.

### Endpoint

```
DELETE https://api.publora.com/media/post/media/:mediaId
```

### Authentication

This endpoint requires an active dashboard session (cookie-based auth). It cannot be called with an API key.

The endpoint verifies media ownership — it filters by the authenticated user's ID, so a user can only delete their own media files.

### Response

```json
{
  "success": true
}
```

On success, the endpoint returns `{ success: true }`.

### Errors

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"Cannot delete media from a post that has already been published or failed"` | The post has already been published or is in a failed state |
| 401 | Unauthorized | No active session |
| 404 | `"Media file not found"` | No media record found for the given `mediaId` |
| 500 | `"Failed to delete media file"` | Internal server error during media deletion |

> **Note:** If S3 deletion fails, the database record is still deleted. This could leave orphaned files in S3.

## File URLs

Uploaded file URLs use the S3 format and are returned directly as `fileUrl` in the response from the `get-upload-url` endpoint. For example:

```
https://brandcraft-media.s3.amazonaws.com/images/1710500000000-product-photo.jpg
```

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
