# YouTube API - Upload Video via REST API

Upload videos to YouTube programmatically using the Publora REST API. A simpler alternative to the official YouTube Data API, YouTube API client libraries, or Google APIs.

## YouTube API Overview

Publora provides a unified REST API for uploading and publishing videos to YouTube with configurable privacy settings (public, unlisted, private) and metadata (title, description). No need to manage Google OAuth flows, handle resumable uploads, or navigate YouTube API quotas.

### Why Use Publora Instead of YouTube Data API?

| Feature | Publora API | YouTube Data API |
|---------|-------------|------------------|
| Authentication | Single API key | Complex Google OAuth 2.0 |
| Video upload | Simple REST endpoint | Resumable upload protocol |
| Quota management | Handled automatically | Manual quota tracking |
| Multi-platform | Post to 11 platforms | YouTube only |
| Setup time | 5 minutes | Hours (Google Cloud setup) |
| Privacy controls | Full support | Full support |

### Keywords: YouTube API, YouTube upload video API, YouTube Data API, upload to YouTube programmatically, YouTube video upload API, YouTube posting API, YouTube REST API, YouTube developer API, YouTube automation API, publish video YouTube API, YouTube content API

## Platform ID Format

```
youtube-{channelId}
```

Where `{channelId}` is your YouTube channel ID assigned during account connection via Google OAuth.

## Requirements

- A YouTube channel connected via Google OAuth through the Publora dashboard
- API key from Publora
- Video content is **required** (YouTube is a video platform)

> **Refresh token lifespan:** YouTube refresh tokens in Publora's system default to approximately 6 months expiry. After that period, the user will need to re-authenticate via the Publora dashboard to reconnect their YouTube channel.

## Supported Content

| Type | Supported | Limits |
|------|-----------|--------|
| Text only | No | YouTube requires a video |
| Images | No | Not supported as standalone posts |
| Videos | Yes | MP4 or MOV format, single video only |

> **Important:** YouTube posts through Publora support only a single video per post. If you attempt to attach multiple videos, the API will reject the request. To publish multiple videos, create separate posts for each.

## Platform-Specific Settings

YouTube supports a `platformSettings` object to control video privacy and title:

```json
{
  "platformSettings": {
    "youtube": {
      "privacy": "public",
      "title": "My Video Title"
    }
  }
}
```

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `privacy` | `"private"`, `"public"`, `"unlisted"` | `"public"` | Video visibility on YouTube |
| `title` | string | `""` (empty string) | Title displayed on the video page. The publisher service may derive a title from the first 70 characters of the post content at publish time, but the API stores an empty string by default at post creation. |

> **Note:** If no title is specified, the API defaults the title to an empty string `""` at post creation. The publisher service may derive a title from the first 70 characters of the post content at publish time. To set a custom title, pass it explicitly via `platformSettings` in the `create-post` request body. The API merges user-provided settings with defaults per platform.

### Privacy Settings

| Value | Description |
|-------|-------------|
| `private` | Only you can view the video |
| `public` | Anyone can search for and view the video |
| `unlisted` | Anyone with the link can view, but it will not appear in search results |

### Title Behavior

If you do not specify a `title` in the platform settings, the API defaults the title to an empty string `""` at post creation. The publisher service may derive a title from the first 70 characters of the post content at publish time. The full `content` is used as the video description. Pass `platformSettings` directly in the `create-post` request body. The API merges user-provided settings with defaults per platform.

## Examples

### Upload a Public Video

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'How to Build a REST API in 10 Minutes - A complete tutorial covering Express.js setup, routing, middleware, error handling, and deployment to production. Perfect for beginners who want to get started with backend development.',
    platforms: ['youtube-UCxxxxxxxx'],
    platformSettings: {
      youtube: {
        privacy: 'public',
        title: 'How to Build a REST API in 10 Minutes'
      }
    }
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
        'content': 'How to Build a REST API in 10 Minutes - A complete tutorial covering Express.js setup, routing, middleware, error handling, and deployment to production. Perfect for beginners who want to get started with backend development.',
        'platforms': ['youtube-UCxxxxxxxx'],
        'platformSettings': {
            'youtube': {
                'privacy': 'public',
                'title': 'How to Build a REST API in 10 Minutes'
            }
        }
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
    "content": "How to Build a REST API in 10 Minutes - A complete tutorial covering Express.js setup, routing, middleware, error handling, and deployment to production. Perfect for beginners who want to get started with backend development.",
    "platforms": ["youtube-UCxxxxxxxx"],
    "platformSettings": {
      "youtube": {
        "privacy": "public",
        "title": "How to Build a REST API in 10 Minutes"
      }
    }
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'How to Build a REST API in 10 Minutes - A complete tutorial covering Express.js setup, routing, middleware, error handling, and deployment to production. Perfect for beginners who want to get started with backend development.',
  platforms: ['youtube-UCxxxxxxxx'],
  platformSettings: {
    youtube: {
      privacy: 'public',
      title: 'How to Build a REST API in 10 Minutes'
    }
  }
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

> **Note:** YouTube requires a video. First create the post, then upload the video using the [media upload workflow](../guides/media-uploads.md) with the returned `postGroupId`.

### Upload an Unlisted Video

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Internal demo recording for the team. This video covers the new dashboard features and upcoming roadmap items. Please do not share this link externally.',
    platforms: ['youtube-UCxxxxxxxx'],
    platformSettings: {
      youtube: {
        privacy: 'unlisted',
        title: 'Internal Demo - Q1 Dashboard Features'
      }
    }
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
        'content': 'Internal demo recording for the team. This video covers the new dashboard features and upcoming roadmap items. Please do not share this link externally.',
        'platforms': ['youtube-UCxxxxxxxx'],
        'platformSettings': {
            'youtube': {
                'privacy': 'unlisted',
                'title': 'Internal Demo - Q1 Dashboard Features'
            }
        }
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
    "content": "Internal demo recording for the team. This video covers the new dashboard features and upcoming roadmap items. Please do not share this link externally.",
    "platforms": ["youtube-UCxxxxxxxx"],
    "platformSettings": {
      "youtube": {
        "privacy": "unlisted",
        "title": "Internal Demo - Q1 Dashboard Features"
      }
    }
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Internal demo recording for the team. This video covers the new dashboard features and upcoming roadmap items. Please do not share this link externally.',
  platforms: ['youtube-UCxxxxxxxx'],
  platformSettings: {
    youtube: {
      privacy: 'unlisted',
      title: 'Internal Demo - Q1 Dashboard Features'
    }
  }
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

console.log(response.data);
```

### Upload with Auto-Generated Title

If you omit the `title` setting, the API defaults the title to an empty string `""` at post creation. The publisher service may derive a title from the first 70 characters of the post content at publish time. To set a custom title, include it in `platformSettings` when calling `create-post`.

**JavaScript (fetch)**

```javascript
const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  },
  body: JSON.stringify({
    content: 'Weekly Vlog: What I learned shipping 3 features in 5 days. This week was intense but incredibly productive. We managed to ship the new analytics dashboard, the team collaboration feature, and a complete redesign of the onboarding flow.',
    platforms: ['youtube-UCxxxxxxxx']
  })
});

const data = await response.json();
// Note: title defaults to an empty string at post creation — the publisher service may derive one from content at publish time. Include platformSettings.youtube.title to set a custom one
// Description will be the full content
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
        'content': 'Weekly Vlog: What I learned shipping 3 features in 5 days. This week was intense but incredibly productive. We managed to ship the new analytics dashboard, the team collaboration feature, and a complete redesign of the onboarding flow.',
        'platforms': ['youtube-UCxxxxxxxx']
    }
)

data = response.json()
# Note: title defaults to an empty string at post creation — the publisher service may derive one from content at publish time. Include platformSettings.youtube.title to set a custom one
print(data)
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "content": "Weekly Vlog: What I learned shipping 3 features in 5 days. This week was intense but incredibly productive. We managed to ship the new analytics dashboard, the team collaboration feature, and a complete redesign of the onboarding flow.",
    "platforms": ["youtube-UCxxxxxxxx"]
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const response = await axios.post('https://api.publora.com/api/v1/create-post', {
  content: 'Weekly Vlog: What I learned shipping 3 features in 5 days. This week was intense but incredibly productive. We managed to ship the new analytics dashboard, the team collaboration feature, and a complete redesign of the onboarding flow.',
  platforms: ['youtube-UCxxxxxxxx']
}, {
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  }
});

// Note: title defaults to an empty string at post creation — the publisher service may derive one from content at publish time. Include platformSettings.youtube.title to set a custom one
console.log(response.data);
```

## Platform Quirks

- **Video only**: YouTube does not support text-only or image-only posts through the API. A video file must always be included.
- **MP4 or MOV recommended**: Publora validates video format server-side using the platform limits configuration (`postValidationService.js` calls `isVideoFormatSupported(platform, mimeType)` for all platforms including YouTube). YouTube recommends MP4 or MOV for best compatibility. If you upload an unsupported format, Publora will reject it before it reaches the YouTube API.
- **Single video only**: Publora supports uploading one video per post. If you attempt to attach multiple videos, the API will reject the request with an error.
- **Default title is empty string**: When no `title` is specified in platform settings, the API defaults the title to an empty string `""` at post creation. The publisher service may derive a title from the first 70 characters of the post content at publish time. It is recommended to explicitly set the title via `platformSettings` in the `create-post` request for best results.
- **Content becomes description**: The full `content` text is used as the YouTube video description, while the `title` (or auto-generated title) is used as the video title.
- **Processing time**: After upload, YouTube processes the video before it becomes available. This can take from a few seconds to several minutes depending on video length and resolution.
- **Daily upload quota**: YouTube enforces daily upload quotas. If you hit the quota, Publora will return the error from the YouTube API.
- **Privacy changes**: You can upload as `private` or `unlisted` first, then manually change the privacy to `public` later through YouTube Studio.

## API Limits

### Text Limits

| Element | Limit |
|---------|-------|
| Video title | 100 characters |
| Video description | 1,000 characters (frontend validation only; YouTube allows up to 5,000) |
| Default title | Empty string `""` at post creation (the publisher service may derive a title from the first 70 characters of post content at publish time) |
| Visible description | First 150 characters (shown before "Show more") |

### Media Limits

| Media Type | Max Size | Max Duration | Supported Formats |
|------------|----------|--------------|-------------------|
| Videos | 512 MB (Publora server limit; YouTube natively allows up to 256 GB) | 12 hours | MP4, MOV (Publora accepts MP4 and MOV; YouTube natively accepts additional formats like AVI, WebM) |
| Images | Not supported | - | - |

### Additional Notes

- YouTube is a video-only platform; images cannot be posted as standalone content
- The first 150 characters of the description are visible without clicking "Show more"
- Video processing time varies based on length and resolution
- The size (256 GB) and duration (12 hours) limits listed above are YouTube's native limits. Publora does not independently enforce these; the YouTube API will reject files that exceed them.
- Publora enforces a 1,000-character limit on video descriptions via frontend validation only. The API itself does not enforce this limit, so API users can send up to YouTube's native 5,000-character limit.

---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
