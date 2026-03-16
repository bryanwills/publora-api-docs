# Media Uploads

This guide covers how to upload images and videos to Publora using pre-signed S3 URLs, and how to attach media to your scheduled posts.

## How It Works

Publora uses a **pre-signed S3 URL workflow** for media uploads. This means your files are uploaded directly to cloud storage without passing through the Publora API server, ensuring fast and reliable transfers.

The workflow has three steps:

```
1. Create a post group first           POST /api/v1/create-post
2. Request a pre-signed upload URL     POST /api/v1/get-upload-url  (with postGroupId)
3. Upload the file directly to S3      PUT {uploadUrl}
```

Media is automatically attached to the post group via the `postGroupId` you provide when requesting the upload URL.

> **Note:** Pre-signed upload URLs expire after **1 hour**. If you do not complete the upload within that time, request a new URL via `POST /api/v1/get-upload-url`.

### Supported Formats

| Type | Formats |
|---|---|
| Images | JPEG, PNG, GIF, WebP |
| Videos | MP4, MOV |

### Limits

- **Maximum file size:** 512 MB per file. All uploads go through a 512 MB multer limit on the Publora server regardless of platform. While some platforms natively support larger files (e.g., YouTube 256 GB, Facebook 2 GB, Telegram 2 GB), Publora caps all uploads at 512 MB. Note: this limit is not enforced server-side via a `ContentLengthRange` condition on the presigned URL, but the 512 MB multer middleware limit applies to the upload URL request.
- **Per post:** Up to **4 images** OR **1 video** (not both). Note: per-post media limits are validated at scheduling time (when the post moves to `scheduled` status), not at upload time
- **Instagram carousels:** Up to **10 images** (exception to the 4-image limit)
- **Threads carousels:** Up to **10 images** (video items in carousels are not currently supported by Publora; standalone video posts work normally)

### Automatic Processing

- **WebP images** are automatically converted to JPEG for platforms that do not support WebP natively.
- **Video metadata** is automatically extracted upon upload, including: resolution, codec, FPS, bitrate, duration, and aspect ratio. This metadata is used for platform-specific validation (e.g., TikTok FPS requirements).

## Examples

### Upload an Image and Create a Post

> **Important:** Create the post as a **draft** (omit `scheduledTime`), upload media, then schedule via update-post. This ensures media is fully uploaded before the scheduler processes your post.

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
    content: 'Check out our latest product!',
    platforms: ['twitter-123', 'linkedin-ABC', 'instagram-456']
    // No scheduledTime = draft
  })
});

const { postGroupId } = await postResponse.json();

// Step 2: Get a pre-signed upload URL
const uploadUrlResponse = await fetch(`${BASE_URL}/get-upload-url`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    fileName: 'product-photo.jpg',
    contentType: 'image/jpeg',
    type: 'image',
    postGroupId: postGroupId
  })
});

const { success, uploadUrl, fileUrl, mediaId } = await uploadUrlResponse.json();
// Response: { success: true, uploadUrl: "...", fileUrl: "...", mediaId: "..." }

// Step 3: Upload the file directly to S3
const fileBuffer = await fs.promises.readFile('./product-photo.jpg');
await fetch(uploadUrl, {
  method: 'PUT',
  headers: { 'Content-Type': 'image/jpeg' },
  body: fileBuffer
});

console.log('Image uploaded:', fileUrl);

// Step 4: Schedule the post
await fetch(`${BASE_URL}/update-post/${postGroupId}`, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    status: 'scheduled',
    scheduledTime: '2026-03-15T14:30:00.000Z'
  })
});

console.log('Post scheduled!');
```

**Python (requests)**

```python
import requests

API_URL = 'https://api.publora.com/api/v1'
HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
}

# Step 1: Create a draft post (no scheduledTime)
post_response = requests.post(
    f'{API_URL}/create-post',
    headers=HEADERS,
    json={
        'content': 'Check out our latest product!',
        'platforms': ['twitter-123', 'linkedin-ABC', 'instagram-456']
        # No scheduledTime = draft
    }
)

post_group_id = post_response.json()['postGroupId']

# Step 2: Get a pre-signed upload URL
upload_url_response = requests.post(
    f'{API_URL}/get-upload-url',
    headers=HEADERS,
    json={
        'fileName': 'product-photo.jpg',
        'contentType': 'image/jpeg',
        'type': 'image',
        'postGroupId': post_group_id
    }
)

upload_data = upload_url_response.json()

# Step 3: Upload the file directly to S3
with open('./product-photo.jpg', 'rb') as f:
    requests.put(
        upload_data['uploadUrl'],
        headers={'Content-Type': 'image/jpeg'},
        data=f.read()
    )

print(f"Image uploaded: {upload_data['fileUrl']}")

# Step 4: Schedule the post
requests.put(
    f'{API_URL}/update-post/{post_group_id}',
    headers=HEADERS,
    json={
        'status': 'scheduled',
        'scheduledTime': '2026-03-15T14:30:00.000Z'
    }
)

print('Post scheduled!')
```

**cURL**

```bash
API_KEY="YOUR_API_KEY"

# Step 1: Create a draft post (no scheduledTime)
POST_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Check out our latest product!",
    "platforms": ["twitter-123", "linkedin-ABC", "instagram-456"]
  }')

POST_GROUP_ID=$(echo "$POST_RESPONSE" | jq -r '.postGroupId')

# Step 2: Get a pre-signed upload URL
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

# Step 3: Upload the file directly to S3
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @./product-photo.jpg

# Step 4: Schedule the post
curl -X PUT "https://api.publora.com/api/v1/update-post/$POST_GROUP_ID" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "status": "scheduled",
    "scheduledTime": "2026-03-15T14:30:00.000Z"
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');
const fs = require('fs');

const api = axios.create({
  baseURL: 'https://api.publora.com/api/v1',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

// Step 1: Create a draft post (no scheduledTime)
const { data: postData } = await api.post('/create-post', {
  content: 'Check out our latest product!',
  platforms: ['twitter-123', 'linkedin-ABC', 'instagram-456']
  // No scheduledTime = draft
});

// Step 2: Get a pre-signed upload URL
const { data: uploadData } = await api.post('/get-upload-url', {
  fileName: 'product-photo.jpg',
  contentType: 'image/jpeg',
  type: 'image',
  postGroupId: postData.postGroupId
});

// Step 3: Upload the file directly to S3
const fileBuffer = fs.readFileSync('./product-photo.jpg');
await axios.put(uploadData.uploadUrl, fileBuffer, {
  headers: { 'Content-Type': 'image/jpeg' }
});

console.log('Image uploaded:', uploadData.fileUrl);

// Step 4: Schedule the post
await api.put(`/update-post/${postData.postGroupId}`, {
  status: 'scheduled',
  scheduledTime: '2026-03-15T14:30:00.000Z'
});

console.log('Post scheduled!');
```

---

### Upload a Video and Create a Post

> **Important:** Create the post as a **draft** first, upload the video, then schedule.

**JavaScript (fetch)**

```javascript
const fs = require('fs');
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
    content: 'Watch our latest promo video!',
    platforms: ['twitter-123', 'tiktok-789', 'youtube-012']
    // No scheduledTime = draft
  })
});

const { postGroupId } = await postResponse.json();

// Step 2: Get a pre-signed upload URL for the video
const uploadUrlResponse = await fetch(`${BASE_URL}/get-upload-url`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    fileName: 'promo-video.mp4',
    contentType: 'video/mp4',
    type: 'video',
    postGroupId: postGroupId
  })
});

const { uploadUrl } = await uploadUrlResponse.json();

// Step 3: Upload the video to S3
const videoBuffer = await fs.promises.readFile('./promo-video.mp4');
await fetch(uploadUrl, {
  method: 'PUT',
  headers: { 'Content-Type': 'video/mp4' },
  body: videoBuffer
});

console.log('Video uploaded:', postGroupId);

// Step 4: Schedule the post
await fetch(`${BASE_URL}/update-post/${postGroupId}`, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': API_KEY
  },
  body: JSON.stringify({
    status: 'scheduled',
    scheduledTime: '2026-03-15T16:00:00.000Z'
  })
});

console.log('Post scheduled!');
```

**Python (requests)**

```python
import requests

API_URL = 'https://api.publora.com/api/v1'
HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
}

# Step 1: Create a draft post (no scheduledTime)
post_response = requests.post(
    f'{API_URL}/create-post',
    headers=HEADERS,
    json={
        'content': 'Watch our latest promo video!',
        'platforms': ['twitter-123', 'tiktok-789', 'youtube-012']
        # No scheduledTime = draft
    }
)

post_group_id = post_response.json()['postGroupId']

# Step 2: Get a pre-signed upload URL
upload_response = requests.post(
    f'{API_URL}/get-upload-url',
    headers=HEADERS,
    json={
        'fileName': 'promo-video.mp4',
        'contentType': 'video/mp4',
        'type': 'video',
        'postGroupId': post_group_id
    }
)

upload_url = upload_response.json()['uploadUrl']

# Step 3: Upload the video to S3
with open('./promo-video.mp4', 'rb') as f:
    requests.put(upload_url, headers={'Content-Type': 'video/mp4'}, data=f.read())

print(f"Video uploaded: {post_group_id}")

# Step 4: Schedule the post
requests.put(
    f'{API_URL}/update-post/{post_group_id}',
    headers=HEADERS,
    json={
        'status': 'scheduled',
        'scheduledTime': '2026-03-15T16:00:00.000Z'
    }
)

print('Post scheduled!')
```

**cURL**

```bash
API_KEY="YOUR_API_KEY"

# Step 1: Create a draft post (no scheduledTime)
POST_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Watch our latest promo video!",
    "platforms": ["twitter-123", "tiktok-789", "youtube-012"]
  }')

POST_GROUP_ID=$(echo "$POST_RESPONSE" | jq -r '.postGroupId')

# Step 2: Get upload URL
UPLOAD_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/get-upload-url \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d "{
    \"fileName\": \"promo-video.mp4\",
    \"contentType\": \"video/mp4\",
    \"type\": \"video\",
    \"postGroupId\": \"$POST_GROUP_ID\"
  }")

UPLOAD_URL=$(echo "$UPLOAD_RESPONSE" | jq -r '.uploadUrl')

# Step 3: Upload video to S3
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: video/mp4" \
  --data-binary @./promo-video.mp4

# Step 4: Schedule the post
curl -X PUT "https://api.publora.com/api/v1/update-post/$POST_GROUP_ID" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "status": "scheduled",
    "scheduledTime": "2026-03-15T16:00:00.000Z"
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');
const fs = require('fs');

const api = axios.create({
  baseURL: 'https://api.publora.com/api/v1',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

// Step 1: Create a draft post (no scheduledTime)
const { data: postData } = await api.post('/create-post', {
  content: 'Watch our latest promo video!',
  platforms: ['twitter-123', 'tiktok-789', 'youtube-012']
  // No scheduledTime = draft
});

// Step 2: Get upload URL
const { data: uploadData } = await api.post('/get-upload-url', {
  fileName: 'promo-video.mp4',
  contentType: 'video/mp4',
  type: 'video',
  postGroupId: postData.postGroupId
});

// Step 3: Upload video
const videoBuffer = fs.readFileSync('./promo-video.mp4');
await axios.put(uploadData.uploadUrl, videoBuffer, {
  headers: { 'Content-Type': 'video/mp4' },
  maxContentLength: 512 * 1024 * 1024, // 512 MB
  maxBodyLength: 512 * 1024 * 1024
});

console.log('Video uploaded:', postData.postGroupId);

// Step 4: Schedule the post
await api.put(`/update-post/${postData.postGroupId}`, {
  status: 'scheduled',
  scheduledTime: '2026-03-15T16:00:00.000Z'
});

console.log('Post scheduled!');
```

---

### Upload Multiple Images for a Carousel Post

You can attach up to 4 images to a single post (or up to 10 for Instagram carousels). Each image requires its own upload URL.

> **Important:** Create the post as a **draft** first, upload all images, then schedule.

**JavaScript (fetch)**

```javascript
const fs = require('fs');
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
    content: 'Our product lineup for 2026 -- swipe to see all!',
    platforms: ['twitter-123', 'linkedin-ABC', 'instagram-456']
    // No scheduledTime = draft
  })
});

const { postGroupId } = await postResponse.json();

// Step 2: Upload each image
const images = [
  { path: './slide1.jpg', name: 'slide1.jpg', type: 'image/jpeg' },
  { path: './slide2.png', name: 'slide2.png', type: 'image/png' },
  { path: './slide3.jpg', name: 'slide3.jpg', type: 'image/jpeg' },
  { path: './slide4.jpg', name: 'slide4.jpg', type: 'image/jpeg' }
];

for (const image of images) {
  // Get upload URL for each image
  const uploadUrlResponse = await fetch(`${BASE_URL}/get-upload-url`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': API_KEY
    },
    body: JSON.stringify({
      fileName: image.name,
      contentType: image.type,
      type: 'image',
      postGroupId: postGroupId
    })
  });

  const { uploadUrl } = await uploadUrlResponse.json();

  // Upload the file to S3
  const fileBuffer = await fs.promises.readFile(image.path);
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': image.type },
    body: fileBuffer
  });

  console.log(`Uploaded ${image.name}`);
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
    scheduledTime: '2026-03-15T12:00:00.000Z'
  })
});

console.log('Carousel post scheduled!');
```

**Python (requests)**

```python
import requests

API_URL = 'https://api.publora.com/api/v1'
HEADERS = {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
}

# Step 1: Create a draft post (no scheduledTime)
post_response = requests.post(
    f'{API_URL}/create-post',
    headers=HEADERS,
    json={
        'content': 'Our product lineup for 2026 -- swipe to see all!',
        'platforms': ['twitter-123', 'linkedin-ABC', 'instagram-456']
        # No scheduledTime = draft
    }
)

post_group_id = post_response.json()['postGroupId']

# Step 2: Upload each image
images = [
    {'path': './slide1.jpg', 'name': 'slide1.jpg', 'type': 'image/jpeg'},
    {'path': './slide2.png', 'name': 'slide2.png', 'type': 'image/png'},
    {'path': './slide3.jpg', 'name': 'slide3.jpg', 'type': 'image/jpeg'},
    {'path': './slide4.jpg', 'name': 'slide4.jpg', 'type': 'image/jpeg'},
]

for image in images:
    # Get upload URL
    upload_response = requests.post(
        f'{API_URL}/get-upload-url',
        headers=HEADERS,
        json={
            'fileName': image['name'],
            'contentType': image['type'],
            'type': 'image',
            'postGroupId': post_group_id
        }
    )

    upload_data = upload_response.json()

    # Upload to S3
    with open(image['path'], 'rb') as f:
        requests.put(
            upload_data['uploadUrl'],
            headers={'Content-Type': image['type']},
            data=f.read()
        )

    print(f"Uploaded {image['name']}")

# Step 3: Schedule the post
requests.put(
    f'{API_URL}/update-post/{post_group_id}',
    headers=HEADERS,
    json={
        'status': 'scheduled',
        'scheduledTime': '2026-03-15T12:00:00.000Z'
    }
)

print('Carousel post scheduled!')
```

**cURL**

```bash
API_KEY="YOUR_API_KEY"

# Step 1: Create a draft post (no scheduledTime)
POST_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "content": "Our product lineup for 2026 -- swipe to see all!",
    "platforms": ["twitter-123", "linkedin-ABC", "instagram-456"]
  }')

POST_GROUP_ID=$(echo "$POST_RESPONSE" | jq -r '.postGroupId')

# Step 2: Upload 4 images
for FILE in slide1.jpg slide2.png slide3.jpg slide4.jpg; do
  CONTENT_TYPE="image/jpeg"
  if [[ "$FILE" == *.png ]]; then
    CONTENT_TYPE="image/png"
  fi

  UPLOAD_RESPONSE=$(curl -s -X POST https://api.publora.com/api/v1/get-upload-url \
    -H "Content-Type: application/json" \
    -H "x-publora-key: $API_KEY" \
    -d "{
      \"fileName\": \"$FILE\",
      \"contentType\": \"$CONTENT_TYPE\",
      \"type\": \"image\",
      \"postGroupId\": \"$POST_GROUP_ID\"
    }")

  UPLOAD_URL=$(echo "$UPLOAD_RESPONSE" | jq -r '.uploadUrl')

  curl -s -X PUT "$UPLOAD_URL" \
    -H "Content-Type: $CONTENT_TYPE" \
    --data-binary @"./$FILE"

  echo "Uploaded $FILE"
done

# Step 3: Schedule the post
curl -X PUT "https://api.publora.com/api/v1/update-post/$POST_GROUP_ID" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: $API_KEY" \
  -d '{
    "status": "scheduled",
    "scheduledTime": "2026-03-15T12:00:00.000Z"
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');
const fs = require('fs');

const api = axios.create({
  baseURL: 'https://api.publora.com/api/v1',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

// Step 1: Create a draft post (no scheduledTime)
const { data: postData } = await api.post('/create-post', {
  content: 'Our product lineup for 2026 -- swipe to see all!',
  platforms: ['twitter-123', 'linkedin-ABC', 'instagram-456']
  // No scheduledTime = draft
});

// Step 2: Upload each image
const images = [
  { path: './slide1.jpg', name: 'slide1.jpg', type: 'image/jpeg' },
  { path: './slide2.png', name: 'slide2.png', type: 'image/png' },
  { path: './slide3.jpg', name: 'slide3.jpg', type: 'image/jpeg' },
  { path: './slide4.jpg', name: 'slide4.jpg', type: 'image/jpeg' }
];

for (const image of images) {
  const { data: uploadData } = await api.post('/get-upload-url', {
    fileName: image.name,
    contentType: image.type,
    type: 'image',
    postGroupId: postData.postGroupId
  });

  const fileBuffer = fs.readFileSync(image.path);
  await axios.put(uploadData.uploadUrl, fileBuffer, {
    headers: { 'Content-Type': image.type }
  });

  console.log(`Uploaded ${image.name}`);
}

// Step 3: Schedule the post
await api.put(`/update-post/${postData.postGroupId}`, {
  status: 'scheduled',
  scheduledTime: '2026-03-15T12:00:00.000Z'
});

console.log('Carousel post scheduled!');
```

## Best Practices

1. **Validate file size before uploading.** The documented maximum is 512 MB per file. While the server does not enforce this via a presigned URL condition, exceeding this size may cause issues with platform processing. Always check the size client-side.

2. **Use the correct `contentType`.** The `contentType` you pass to `get-upload-url` must match the `Content-Type` header you send when uploading to S3. A mismatch will cause the upload to fail.

3. **Prefer JPEG over WebP.** While WebP is supported and will be auto-converted, starting with JPEG avoids the conversion step and ensures consistent quality across all platforms.

4. **Upload images in parallel when possible.** For carousel posts, you can request all 4 upload URLs at once and upload concurrently to speed things up.

5. **For large video files, consider streaming the upload.** Instead of reading the entire file into memory, use a stream-based approach for files approaching the 512 MB limit.

## Common Issues

| Problem | Cause | Solution |
|---|---|---|
| S3 upload returns `403 Forbidden` | Pre-signed URL expired or `Content-Type` mismatch | Request a fresh upload URL and ensure the `Content-Type` header matches exactly |
| `400` when uploading media | Missing required fields (`fileName`, `contentType`, `postGroupId`) | Ensure all required fields are provided. Note: `type` (`"image"` or `"video"`) is effectively required -- while the API does not explicitly validate it, omitting it causes a downstream error during processing. Always include it. |
| WebP image looks different after posting | Auto-conversion to JPEG for incompatible platforms | Upload as JPEG directly if quality consistency is critical |
| Video post fails on TikTok | Video does not meet TikTok requirements (FPS, format, duration) | Check video metadata -- ensure proper FPS, supported codec, and acceptable duration |
| Upload is slow for large files | File being read entirely into memory | Use streaming upload for large video files |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
