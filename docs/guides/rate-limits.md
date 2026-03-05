# Rate Limits and Optimal Posting Times

This guide covers platform-specific rate limits, Publora API rate limits, and strategies for scheduling posts at optimal engagement times.

## Publora API Rate Limits

| Plan | Requests/Minute | Requests/Hour | Concurrent Posts |
|------|-----------------|---------------|------------------|
| Free Trial | 30 | 500 | 5 pending |
| Starter | 60 | 2,000 | 50 pending |
| Pro | 120 | 10,000 | 500 pending |
| Enterprise | Custom | Custom | Unlimited |

### Rate Limit Headers

Every API response includes rate limit information:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1709913600
```

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests per minute for your plan |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the limit resets |

### Rate Limit Response

When you exceed the rate limit, you'll receive:

```json
{
  "success": false,
  "error": "Rate limit exceeded",
  "retryAfter": 45
}
```

**HTTP Status:** `429 Too Many Requests`

## Platform-Specific Rate Limits

Each social media platform has its own posting limits. Publora handles these automatically, but understanding them helps you design better scheduling strategies.

| Platform | Posts/Day | Posts/Hour | Notes |
|----------|-----------|------------|-------|
| **X/Twitter** | 2,400 tweets | 300 tweets | Per account; threading counts as multiple |
| **LinkedIn** | 100 posts | 25 posts | Per organization page |
| **Instagram** | 25 posts | 10 posts | Feed posts only; Stories unlimited |
| **Threads** | 250 posts | 50 posts | Meta's Thread-specific limits |
| **TikTok** | 50 videos | 10 videos | Video uploads only |
| **YouTube** | 50 videos | 10 videos | Per channel |
| **Facebook** | 50 posts | 25 posts | Per page |
| **Bluesky** | 1,666 posts | 100 posts | Per account |
| **Mastodon** | 300 posts | 30 posts | Instance-dependent |
| **Telegram** | Unlimited | 20 messages/sec | Channel/group dependent |

### How Publora Handles Platform Limits

1. **Automatic Queuing** - If a platform rate limit is hit, Publora queues the post and retries automatically
2. **Smart Distribution** - When scheduling many posts, Publora distributes them to avoid hitting limits
3. **Error Reporting** - If a post fails due to platform limits, status shows `failed` with the reason

## Examples

### JavaScript - Rate Limit Aware Client

```javascript
class RateLimitedPubloraClient {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = 'https://api.publora.com/api/v1';
    this.rateLimitRemaining = Infinity;
    this.rateLimitReset = 0;
  }

  async request(endpoint, options = {}) {
    // Wait if we've exhausted rate limit
    if (this.rateLimitRemaining <= 0) {
      const waitTime = (this.rateLimitReset * 1000) - Date.now();
      if (waitTime > 0) {
        console.log(`Rate limited. Waiting ${Math.ceil(waitTime / 1000)}s...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      }
    }

    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      ...options,
      headers: {
        'x-publora-key': this.apiKey,
        'Content-Type': 'application/json',
        ...options.headers
      }
    });

    // Update rate limit tracking
    this.rateLimitRemaining = parseInt(response.headers.get('X-RateLimit-Remaining') || '60');
    this.rateLimitReset = parseInt(response.headers.get('X-RateLimit-Reset') || '0');

    if (response.status === 429) {
      const data = await response.json();
      console.log(`Rate limited. Retry after ${data.retryAfter}s`);
      await new Promise(resolve => setTimeout(resolve, data.retryAfter * 1000));
      return this.request(endpoint, options); // Retry
    }

    return response.json();
  }

  async createPost(content, platforms, scheduledTime) {
    return this.request('/create-post', {
      method: 'POST',
      body: JSON.stringify({ content, platforms, scheduledTime })
    });
  }

  async listPosts(params = {}) {
    const query = new URLSearchParams(params).toString();
    return this.request(`/list-posts?${query}`);
  }
}

// Usage
const client = new RateLimitedPubloraClient('YOUR_API_KEY');

// Client automatically handles rate limits
for (let i = 0; i < 100; i++) {
  await client.createPost(
    `Post ${i + 1}`,
    ['twitter-123'],
    new Date(Date.now() + i * 3600000).toISOString()
  );
}
```

### Python - Rate Limit Handler with Exponential Backoff

```python
import requests
import time
from typing import Dict, Any, Optional
from functools import wraps

class RateLimitedPubloraClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = 'https://api.publora.com/api/v1'
        self.rate_limit_remaining = float('inf')
        self.rate_limit_reset = 0

    def _request(
        self,
        method: str,
        endpoint: str,
        data: Optional[Dict] = None,
        params: Optional[Dict] = None,
        max_retries: int = 3
    ) -> Dict[str, Any]:
        """Make a request with automatic rate limit handling."""

        # Wait if rate limited
        if self.rate_limit_remaining <= 0:
            wait_time = self.rate_limit_reset - time.time()
            if wait_time > 0:
                print(f"Rate limited. Waiting {wait_time:.1f}s...")
                time.sleep(wait_time)

        headers = {
            'x-publora-key': self.api_key,
            'Content-Type': 'application/json'
        }

        for attempt in range(max_retries):
            response = requests.request(
                method,
                f'{self.base_url}{endpoint}',
                headers=headers,
                json=data,
                params=params
            )

            # Update rate limit info
            self.rate_limit_remaining = int(
                response.headers.get('X-RateLimit-Remaining', 60)
            )
            self.rate_limit_reset = int(
                response.headers.get('X-RateLimit-Reset', 0)
            )

            if response.status_code == 429:
                retry_after = response.json().get('retryAfter', 60)
                backoff = retry_after * (2 ** attempt)  # Exponential backoff
                print(f"Rate limited. Waiting {backoff}s (attempt {attempt + 1})")
                time.sleep(backoff)
                continue

            response.raise_for_status()
            return response.json()

        raise Exception("Max retries exceeded")

    def create_post(
        self,
        content: str,
        platforms: list,
        scheduled_time: Optional[str] = None
    ) -> Dict[str, Any]:
        data = {'content': content, 'platforms': platforms}
        if scheduled_time:
            data['scheduledTime'] = scheduled_time
        return self._request('POST', '/create-post', data=data)

    def list_posts(self, **params) -> Dict[str, Any]:
        return self._request('GET', '/list-posts', params=params)


# Usage
client = RateLimitedPubloraClient('YOUR_API_KEY')

# Bulk create with automatic rate limiting
from datetime import datetime, timedelta, timezone

base_time = datetime.now(timezone.utc) + timedelta(hours=1)

for i in range(100):
    scheduled = (base_time + timedelta(hours=i)).isoformat()
    result = client.create_post(
        f"Scheduled post {i + 1}",
        ['twitter-123', 'linkedin-ABC'],
        scheduled
    )
    print(f"Created post {i + 1}: {result['postGroupId']}")
```

## Peak Engagement Times

Optimal posting times vary by platform and audience. Use these guidelines as a starting point, then analyze your own analytics.

### Platform-Specific Best Times (UTC)

| Platform | Best Days | Best Hours (UTC) | Peak Engagement |
|----------|-----------|------------------|-----------------|
| **X/Twitter** | Tue-Thu | 13:00-16:00 | Wednesday 12:00 |
| **LinkedIn** | Tue-Thu | 10:00-12:00 | Tuesday 10:00 |
| **Instagram** | Mon, Wed | 11:00-14:00, 19:00-21:00 | Wednesday 11:00 |
| **Threads** | Tue-Thu | 12:00-15:00 | Wednesday 13:00 |
| **TikTok** | Tue-Thu | 15:00-21:00 | Thursday 19:00 |
| **YouTube** | Thu-Sat | 14:00-18:00 | Friday 15:00 |
| **Facebook** | Wed-Fri | 13:00-16:00 | Wednesday 13:00 |
| **Bluesky** | Mon-Fri | 14:00-17:00 | Tuesday 15:00 |

### JavaScript - Optimal Time Scheduler

```javascript
const OPTIMAL_TIMES = {
  twitter: {
    bestDays: [2, 3, 4], // Tue, Wed, Thu (0 = Sunday)
    bestHours: [13, 14, 15, 16],
    peakDay: 3,
    peakHour: 12
  },
  linkedin: {
    bestDays: [2, 3, 4],
    bestHours: [10, 11, 12],
    peakDay: 2,
    peakHour: 10
  },
  instagram: {
    bestDays: [1, 3], // Mon, Wed
    bestHours: [11, 12, 13, 14, 19, 20, 21],
    peakDay: 3,
    peakHour: 11
  },
  threads: {
    bestDays: [2, 3, 4],
    bestHours: [12, 13, 14, 15],
    peakDay: 3,
    peakHour: 13
  },
  tiktok: {
    bestDays: [2, 3, 4],
    bestHours: [15, 16, 17, 18, 19, 20, 21],
    peakDay: 4,
    peakHour: 19
  },
  facebook: {
    bestDays: [3, 4, 5], // Wed, Thu, Fri
    bestHours: [13, 14, 15, 16],
    peakDay: 3,
    peakHour: 13
  }
};

function getNextOptimalTime(platform, fromDate = new Date()) {
  const config = OPTIMAL_TIMES[platform];
  if (!config) throw new Error(`Unknown platform: ${platform}`);

  const result = new Date(fromDate);
  result.setUTCMinutes(0, 0, 0);

  // Find next best day
  let daysToAdd = 0;
  while (!config.bestDays.includes(result.getUTCDay()) || daysToAdd === 0) {
    result.setUTCDate(result.getUTCDate() + 1);
    daysToAdd++;
    if (daysToAdd > 7) break;
  }

  // Set to best hour
  result.setUTCHours(config.bestHours[0]);

  return result;
}

function getPeakTime(platform, weekStart = new Date()) {
  const config = OPTIMAL_TIMES[platform];
  if (!config) throw new Error(`Unknown platform: ${platform}`);

  const result = new Date(weekStart);
  result.setUTCHours(0, 0, 0, 0);

  // Move to peak day
  const currentDay = result.getUTCDay();
  const daysUntilPeak = (config.peakDay - currentDay + 7) % 7;
  result.setUTCDate(result.getUTCDate() + daysUntilPeak);

  // Set peak hour
  result.setUTCHours(config.peakHour);

  return result;
}

function distributePostsOptimally(platforms, count, startDate = new Date()) {
  const schedule = [];
  const date = new Date(startDate);

  for (let i = 0; i < count; i++) {
    const platform = platforms[i % platforms.length];
    const optimalTime = getNextOptimalTime(platform, date);

    schedule.push({
      index: i,
      platform,
      scheduledTime: optimalTime.toISOString()
    });

    // Move forward for next post
    date.setTime(optimalTime.getTime() + 3600000); // +1 hour minimum gap
  }

  return schedule;
}

// Usage
const schedule = distributePostsOptimally(
  ['twitter', 'linkedin', 'instagram'],
  9, // 3 posts per platform
  new Date()
);

console.log('Optimal posting schedule:');
schedule.forEach(item => {
  console.log(`${item.platform}: ${item.scheduledTime}`);
});
```

### Python - Queue-Based Scheduler with Rate Limits

```python
from datetime import datetime, timedelta, timezone
from typing import List, Dict, Optional
from dataclasses import dataclass
from collections import defaultdict
import heapq

@dataclass
class PlatformConfig:
    posts_per_hour: int
    posts_per_day: int
    best_hours: List[int]  # UTC hours
    best_days: List[int]   # 0=Monday, 6=Sunday

PLATFORM_LIMITS = {
    'twitter': PlatformConfig(
        posts_per_hour=10,
        posts_per_day=50,
        best_hours=[13, 14, 15, 16],
        best_days=[1, 2, 3]  # Tue, Wed, Thu
    ),
    'linkedin': PlatformConfig(
        posts_per_hour=5,
        posts_per_day=20,
        best_hours=[10, 11, 12],
        best_days=[1, 2, 3]
    ),
    'instagram': PlatformConfig(
        posts_per_hour=3,
        posts_per_day=10,
        best_hours=[11, 12, 13, 14, 19, 20, 21],
        best_days=[0, 2]  # Mon, Wed
    ),
    'tiktok': PlatformConfig(
        posts_per_hour=2,
        posts_per_day=10,
        best_hours=[15, 16, 17, 18, 19, 20, 21],
        best_days=[1, 2, 3]
    ),
}


class OptimalScheduler:
    """
    Schedule posts optimally across platforms considering:
    - Platform-specific rate limits
    - Peak engagement times
    - Even distribution across time slots
    """

    def __init__(self):
        self.platform_queues: Dict[str, List] = defaultdict(list)
        self.posts_per_hour: Dict[str, Dict[str, int]] = defaultdict(lambda: defaultdict(int))
        self.posts_per_day: Dict[str, Dict[str, int]] = defaultdict(lambda: defaultdict(int))

    def get_next_optimal_slot(
        self,
        platform: str,
        after: datetime
    ) -> datetime:
        """Find the next available optimal time slot for a platform."""
        config = PLATFORM_LIMITS.get(platform)
        if not config:
            # Unknown platform - just space by 1 hour
            return after + timedelta(hours=1)

        candidate = after.replace(minute=0, second=0, microsecond=0)

        for _ in range(168):  # Check up to 1 week ahead
            candidate += timedelta(hours=1)
            day_key = candidate.strftime('%Y-%m-%d')
            hour_key = candidate.strftime('%Y-%m-%d-%H')

            # Check rate limits
            if self.posts_per_day[platform][day_key] >= config.posts_per_day:
                # Skip to next day
                candidate = candidate.replace(hour=0) + timedelta(days=1)
                continue

            if self.posts_per_hour[platform][hour_key] >= config.posts_per_hour:
                continue

            # Check if this is an optimal time
            if (candidate.weekday() in config.best_days and
                candidate.hour in config.best_hours):
                return candidate

        # If no optimal slot found, return next available
        return after + timedelta(hours=1)

    def schedule_post(
        self,
        platform: str,
        after: datetime = None
    ) -> datetime:
        """Schedule a single post and return its time."""
        if after is None:
            after = datetime.now(timezone.utc)

        scheduled_time = self.get_next_optimal_slot(platform, after)

        # Update counters
        day_key = scheduled_time.strftime('%Y-%m-%d')
        hour_key = scheduled_time.strftime('%Y-%m-%d-%H')
        self.posts_per_day[platform][day_key] += 1
        self.posts_per_hour[platform][hour_key] += 1

        return scheduled_time

    def schedule_batch(
        self,
        posts: List[Dict],
        start_after: datetime = None
    ) -> List[Dict]:
        """
        Schedule multiple posts across platforms optimally.

        Args:
            posts: List of {'content': str, 'platforms': List[str]}
            start_after: Earliest scheduling time

        Returns:
            List of posts with scheduledTime added
        """
        if start_after is None:
            start_after = datetime.now(timezone.utc)

        scheduled_posts = []
        platform_last_time: Dict[str, datetime] = {}

        for post in posts:
            platforms = post['platforms']
            # Find the latest "next optimal time" across all platforms
            scheduled_time = start_after

            for platform in platforms:
                platform_name = platform.split('-')[0]
                after = platform_last_time.get(platform_name, start_after)
                optimal_time = self.schedule_post(platform_name, after)

                if optimal_time > scheduled_time:
                    scheduled_time = optimal_time

                platform_last_time[platform_name] = scheduled_time

            scheduled_posts.append({
                **post,
                'scheduledTime': scheduled_time.isoformat()
            })

        return scheduled_posts


# Usage example
scheduler = OptimalScheduler()

# Posts to schedule
posts = [
    {'content': 'Morning motivation!', 'platforms': ['twitter-123', 'linkedin-ABC']},
    {'content': 'Product update announcement', 'platforms': ['twitter-123', 'linkedin-ABC', 'instagram-456']},
    {'content': 'Behind the scenes', 'platforms': ['instagram-456', 'tiktok-789']},
    {'content': 'Weekly tips thread', 'platforms': ['twitter-123']},
    {'content': 'Customer success story', 'platforms': ['linkedin-ABC']},
]

scheduled = scheduler.schedule_batch(posts)

print("Optimally scheduled posts:")
for post in scheduled:
    print(f"  {post['scheduledTime']}: {post['content'][:30]}...")
    print(f"    Platforms: {post['platforms']}")
```

### Node.js - Complete Rate-Limited Queue System

```javascript
const axios = require('axios');

class PubloraQueueScheduler {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.api = axios.create({
      baseURL: 'https://api.publora.com/api/v1',
      headers: { 'x-publora-key': apiKey }
    });

    this.platformLimits = {
      twitter: { perHour: 10, perDay: 50 },
      linkedin: { perHour: 5, perDay: 20 },
      instagram: { perHour: 3, perDay: 10 },
      tiktok: { perHour: 2, perDay: 10 },
      facebook: { perHour: 5, perDay: 25 },
    };

    this.peakHours = {
      twitter: [13, 14, 15, 16],
      linkedin: [10, 11, 12],
      instagram: [11, 12, 13, 14, 19, 20, 21],
      tiktok: [15, 16, 17, 18, 19, 20, 21],
      facebook: [13, 14, 15, 16],
    };

    this.postCounts = { hourly: {}, daily: {} };
  }

  getPlatformName(platformId) {
    return platformId.split('-')[0];
  }

  getHourKey(date) {
    return date.toISOString().slice(0, 13);
  }

  getDayKey(date) {
    return date.toISOString().slice(0, 10);
  }

  canPostAt(platform, date) {
    const limits = this.platformLimits[platform];
    if (!limits) return true;

    const hourKey = `${platform}-${this.getHourKey(date)}`;
    const dayKey = `${platform}-${this.getDayKey(date)}`;

    const hourlyCount = this.postCounts.hourly[hourKey] || 0;
    const dailyCount = this.postCounts.daily[dayKey] || 0;

    return hourlyCount < limits.perHour && dailyCount < limits.perDay;
  }

  recordPost(platform, date) {
    const hourKey = `${platform}-${this.getHourKey(date)}`;
    const dayKey = `${platform}-${this.getDayKey(date)}`;

    this.postCounts.hourly[hourKey] = (this.postCounts.hourly[hourKey] || 0) + 1;
    this.postCounts.daily[dayKey] = (this.postCounts.daily[dayKey] || 0) + 1;
  }

  isPeakTime(platform, date) {
    const hours = this.peakHours[platform];
    return hours ? hours.includes(date.getUTCHours()) : true;
  }

  findNextOptimalSlot(platforms, after) {
    let candidate = new Date(after);
    candidate.setUTCMinutes(0, 0, 0);

    for (let i = 0; i < 168; i++) { // Check up to 1 week
      candidate = new Date(candidate.getTime() + 3600000); // +1 hour

      const allCanPost = platforms.every(p =>
        this.canPostAt(this.getPlatformName(p), candidate)
      );

      if (!allCanPost) continue;

      const anyPeak = platforms.some(p =>
        this.isPeakTime(this.getPlatformName(p), candidate)
      );

      if (anyPeak) {
        return candidate;
      }
    }

    // Fallback: return next hour
    return new Date(after.getTime() + 3600000);
  }

  async scheduleOptimally(posts) {
    const results = [];
    let lastTime = new Date();

    for (const post of posts) {
      const scheduledTime = this.findNextOptimalSlot(post.platforms, lastTime);

      // Record for rate limiting
      post.platforms.forEach(p => {
        this.recordPost(this.getPlatformName(p), scheduledTime);
      });

      try {
        const { data } = await this.api.post('/create-post', {
          content: post.content,
          platforms: post.platforms,
          scheduledTime: scheduledTime.toISOString(),
          mediaUrls: post.mediaUrls
        });

        results.push({
          success: true,
          postGroupId: data.postGroupId,
          scheduledTime: scheduledTime.toISOString(),
          content: post.content.slice(0, 50)
        });

        console.log(`Scheduled: ${post.content.slice(0, 30)}... at ${scheduledTime.toISOString()}`);
      } catch (error) {
        results.push({
          success: false,
          error: error.response?.data?.message || error.message,
          content: post.content.slice(0, 50)
        });
      }

      lastTime = scheduledTime;

      // Small delay between API calls
      await new Promise(r => setTimeout(r, 100));
    }

    return results;
  }
}

// Usage
const scheduler = new PubloraQueueScheduler('YOUR_API_KEY');

const posts = [
  { content: 'Morning motivation!', platforms: ['twitter-123', 'linkedin-ABC'] },
  { content: 'Product update', platforms: ['twitter-123', 'linkedin-ABC', 'instagram-456'] },
  { content: 'Behind the scenes', platforms: ['instagram-456', 'tiktok-789'] },
];

const results = await scheduler.scheduleOptimally(posts);
console.log(`Scheduled ${results.filter(r => r.success).length} of ${results.length} posts`);
```

## Complete Queue-Based Scheduling System

This section provides a production-ready implementation that combines all the concepts: rate limiting, optimal scheduling, error handling, retry logic, and failed post recovery.

### Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Content Queue  │───►│  Rate Limiter    │───►│  Publora API    │
│  (your posts)   │    │  + Scheduler     │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                              │                        │
                              ▼                        ▼
                       ┌──────────────────┐    ┌─────────────────┐
                       │  Retry Queue     │    │  Status Checker │
                       │  (failed posts)  │    │  (polling)      │
                       └──────────────────┘    └─────────────────┘
```

### JavaScript - Production Queue Scheduler

```javascript
const axios = require('axios');

/**
 * Production-ready queue-based scheduler for Publora API
 * Features:
 * - Platform-specific rate limiting
 * - Peak engagement time optimization
 * - Exponential backoff retry logic
 * - Failed post recovery
 * - Comprehensive error handling
 */
class PubloraQueueScheduler {
  constructor(apiKey, options = {}) {
    this.apiKey = apiKey;
    this.baseUrl = options.baseUrl || 'https://api.publora.com/api/v1';
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;

    // API rate limit tracking
    this.rateLimitRemaining = Infinity;
    this.rateLimitReset = 0;

    // Platform-specific limits (posts per hour / per day)
    this.platformLimits = {
      twitter:   { perHour: 10, perDay: 50 },
      linkedin:  { perHour: 5,  perDay: 20 },
      instagram: { perHour: 3,  perDay: 10 },
      threads:   { perHour: 10, perDay: 50 },
      tiktok:    { perHour: 2,  perDay: 10 },
      facebook:  { perHour: 5,  perDay: 25 },
      bluesky:   { perHour: 10, perDay: 100 },
    };

    // Peak engagement hours (UTC)
    this.peakHours = {
      twitter:   [13, 14, 15, 16],
      linkedin:  [10, 11, 12],
      instagram: [11, 12, 13, 14, 19, 20, 21],
      threads:   [12, 13, 14, 15],
      tiktok:    [15, 16, 17, 18, 19, 20, 21],
      facebook:  [13, 14, 15, 16],
      bluesky:   [14, 15, 16, 17],
    };

    // Track scheduled posts per platform
    this.postCounts = { hourly: {}, daily: {} };

    // Failed posts queue for retry
    this.failedQueue = [];

    // Results tracking
    this.results = [];
  }

  /**
   * Make an API request with rate limit handling and retries
   */
  async request(method, endpoint, data = null) {
    // Wait if API rate limited
    await this.waitForRateLimit();

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        const config = {
          method,
          url: `${this.baseUrl}${endpoint}`,
          headers: {
            'Content-Type': 'application/json',
            'x-publora-key': this.apiKey
          }
        };

        if (data) config.data = data;

        const response = await axios(config);

        // Update rate limit tracking from headers
        this.updateRateLimits(response.headers);

        return { success: true, data: response.data };

      } catch (error) {
        const status = error.response?.status;
        const errorData = error.response?.data;
        const message = errorData?.error || errorData?.message || error.message;

        // Handle rate limit (429)
        if (status === 429) {
          const retryAfter = errorData?.retryAfter || 60;
          console.log(`Rate limited. Waiting ${retryAfter}s...`);
          await this.sleep(retryAfter * 1000);
          continue;
        }

        // Don't retry client errors (4xx) except 429
        if (status >= 400 && status < 500) {
          return {
            success: false,
            error: message,
            status,
            retryable: false
          };
        }

        // Retry server errors (5xx) and network errors
        if (attempt < this.maxRetries) {
          const delay = this.retryDelay * Math.pow(2, attempt);
          console.log(`Error (attempt ${attempt + 1}/${this.maxRetries + 1}). Retrying in ${delay}ms...`);
          await this.sleep(delay);
          continue;
        }

        return {
          success: false,
          error: message,
          status: status || 0,
          retryable: true
        };
      }
    }
  }

  /**
   * Wait if we've hit the API rate limit
   */
  async waitForRateLimit() {
    if (this.rateLimitRemaining <= 1) {
      const waitTime = (this.rateLimitReset * 1000) - Date.now();
      if (waitTime > 0) {
        console.log(`API rate limit reached. Waiting ${Math.ceil(waitTime / 1000)}s...`);
        await this.sleep(waitTime + 100);
      }
    }
  }

  /**
   * Update rate limit tracking from response headers
   */
  updateRateLimits(headers) {
    const remaining = headers['x-ratelimit-remaining'];
    const reset = headers['x-ratelimit-reset'];

    if (remaining !== undefined) {
      this.rateLimitRemaining = parseInt(remaining);
    }
    if (reset !== undefined) {
      this.rateLimitReset = parseInt(reset);
    }
  }

  /**
   * Get platform name from platform ID (e.g., "twitter-123" -> "twitter")
   */
  getPlatformName(platformId) {
    return platformId.split('-')[0];
  }

  /**
   * Check if a platform can accept a post at the given time
   */
  canPostAt(platform, date) {
    const limits = this.platformLimits[platform];
    if (!limits) return true;

    const hourKey = `${platform}-${date.toISOString().slice(0, 13)}`;
    const dayKey = `${platform}-${date.toISOString().slice(0, 10)}`;

    const hourlyCount = this.postCounts.hourly[hourKey] || 0;
    const dailyCount = this.postCounts.daily[dayKey] || 0;

    return hourlyCount < limits.perHour && dailyCount < limits.perDay;
  }

  /**
   * Record a scheduled post for rate limiting
   */
  recordPost(platform, date) {
    const hourKey = `${platform}-${date.toISOString().slice(0, 13)}`;
    const dayKey = `${platform}-${date.toISOString().slice(0, 10)}`;

    this.postCounts.hourly[hourKey] = (this.postCounts.hourly[hourKey] || 0) + 1;
    this.postCounts.daily[dayKey] = (this.postCounts.daily[dayKey] || 0) + 1;
  }

  /**
   * Check if the given time is a peak engagement hour
   */
  isPeakTime(platform, date) {
    const hours = this.peakHours[platform];
    return hours ? hours.includes(date.getUTCHours()) : true;
  }

  /**
   * Find the next optimal time slot considering all platforms
   */
  findNextOptimalSlot(platforms, after) {
    let candidate = new Date(after);
    candidate.setUTCMinutes(0, 0, 0);

    // Search up to 1 week ahead
    for (let i = 0; i < 168; i++) {
      candidate = new Date(candidate.getTime() + 3600000); // +1 hour

      // Check if all platforms can post at this time
      const allCanPost = platforms.every(p =>
        this.canPostAt(this.getPlatformName(p), candidate)
      );

      if (!allCanPost) continue;

      // Prefer peak times
      const anyPeak = platforms.some(p =>
        this.isPeakTime(this.getPlatformName(p), candidate)
      );

      if (anyPeak) {
        return candidate;
      }
    }

    // Fallback: return next hour
    return new Date(after.getTime() + 3600000);
  }

  /**
   * Schedule a batch of posts optimally
   */
  async scheduleBatch(posts, options = {}) {
    const startAfter = options.startAfter || new Date();
    const results = [];
    let lastTime = startAfter;

    console.log(`\nScheduling ${posts.length} posts...`);

    for (let i = 0; i < posts.length; i++) {
      const post = posts[i];
      const scheduledTime = this.findNextOptimalSlot(post.platforms, lastTime);

      // Record for platform rate limiting
      post.platforms.forEach(p => {
        this.recordPost(this.getPlatformName(p), scheduledTime);
      });

      const payload = {
        content: post.content,
        platforms: post.platforms,
        scheduledTime: scheduledTime.toISOString()
      };

      if (post.mediaUrls) payload.mediaUrls = post.mediaUrls;
      if (post.platformSettings) payload.platformSettings = post.platformSettings;

      const response = await this.request('POST', '/create-post', payload);

      const result = {
        index: i,
        content: post.content.slice(0, 50),
        platforms: post.platforms,
        scheduledTime: scheduledTime.toISOString(),
        success: response.success
      };

      if (response.success) {
        result.postGroupId = response.data.postGroupId;
        console.log(`[${i + 1}/${posts.length}] ✓ Scheduled: ${result.content}...`);
      } else {
        result.error = response.error;
        result.retryable = response.retryable;
        console.log(`[${i + 1}/${posts.length}] ✗ Failed: ${response.error}`);

        if (response.retryable) {
          this.failedQueue.push({ ...post, originalIndex: i });
        }
      }

      results.push(result);
      lastTime = scheduledTime;

      // Small delay between API calls
      await this.sleep(100);
    }

    this.results = results;
    return results;
  }

  /**
   * Retry failed posts from the queue
   */
  async retryFailed() {
    if (this.failedQueue.length === 0) {
      console.log('No failed posts to retry.');
      return [];
    }

    console.log(`\nRetrying ${this.failedQueue.length} failed posts...`);

    const toRetry = [...this.failedQueue];
    this.failedQueue = [];

    const results = await this.scheduleBatch(toRetry);

    return results;
  }

  /**
   * Check status of scheduled posts and handle failures
   */
  async checkPostStatuses(postGroupIds) {
    const statuses = [];

    for (const postGroupId of postGroupIds) {
      const response = await this.request('GET', `/get-post/${postGroupId}`);

      if (!response.success) {
        statuses.push({
          postGroupId,
          status: 'error',
          error: response.error
        });
        continue;
      }

      const postGroup = response.data;
      const status = {
        postGroupId,
        status: postGroup.status,
        posts: []
      };

      // Check each platform's status
      for (const post of postGroup.posts || []) {
        const postStatus = {
          platform: post.platform,
          platformId: post.platformId,
          status: post.status
        };

        if (post.status === 'published') {
          postStatus.publishedUrl = post.publishedUrl;
        } else if (post.status === 'failed') {
          postStatus.error = post.error;
        }

        status.posts.push(postStatus);
      }

      statuses.push(status);
    }

    return statuses;
  }

  /**
   * Get summary of scheduling results
   */
  getSummary() {
    const successful = this.results.filter(r => r.success).length;
    const failed = this.results.filter(r => !r.success).length;
    const retryable = this.results.filter(r => !r.success && r.retryable).length;

    return {
      total: this.results.length,
      successful,
      failed,
      retryable,
      failedQueueSize: this.failedQueue.length
    };
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ============================================
// USAGE EXAMPLE
// ============================================

async function main() {
  const scheduler = new PubloraQueueScheduler('YOUR_API_KEY');

  // Define posts to schedule
  const posts = [
    {
      content: 'Morning motivation! Start your week with intention. #MondayMotivation',
      platforms: ['twitter-123456', 'linkedin-ABC123']
    },
    {
      content: 'Product update: We just shipped a major new feature...',
      platforms: ['twitter-123456', 'linkedin-ABC123', 'threads-789012']
    },
    {
      content: 'Behind the scenes of our latest photoshoot',
      platforms: ['instagram-DEF456', 'tiktok-GHI789']
    },
    {
      content: 'Weekly tips thread: 5 things I learned this week...',
      platforms: ['twitter-123456', 'threads-789012']
    },
    {
      content: 'Customer success story: How Company X grew 300%',
      platforms: ['linkedin-ABC123']
    }
  ];

  // Schedule all posts
  const results = await scheduler.scheduleBatch(posts, {
    startAfter: new Date() // Start from now
  });

  // Get summary
  const summary = scheduler.getSummary();
  console.log('\n=== SCHEDULING SUMMARY ===');
  console.log(`Total: ${summary.total}`);
  console.log(`Successful: ${summary.successful}`);
  console.log(`Failed: ${summary.failed}`);
  console.log(`Retryable: ${summary.retryable}`);

  // Retry any failed posts
  if (summary.retryable > 0) {
    console.log('\nRetrying failed posts...');
    await scheduler.retryFailed();
  }

  // Later: Check publishing status
  const successfulIds = results
    .filter(r => r.success)
    .map(r => r.postGroupId);

  if (successfulIds.length > 0) {
    console.log('\n=== POST STATUSES ===');
    // Note: Only check after scheduled time has passed
    const statuses = await scheduler.checkPostStatuses(successfulIds);

    for (const status of statuses) {
      console.log(`\nPost ${status.postGroupId}: ${status.status}`);

      if (status.posts) {
        for (const post of status.posts) {
          if (post.status === 'published') {
            console.log(`  ✓ ${post.platform}: ${post.publishedUrl}`);
          } else if (post.status === 'failed') {
            console.log(`  ✗ ${post.platform}: ${post.error}`);
          } else {
            console.log(`  ○ ${post.platform}: ${post.status}`);
          }
        }
      }
    }
  }
}

main().catch(console.error);
```

### Python - Production Queue Scheduler

```python
import requests
import time
from datetime import datetime, timedelta, timezone
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from collections import defaultdict


@dataclass
class ScheduleResult:
    """Result of a scheduling attempt"""
    index: int
    content: str
    platforms: List[str]
    scheduled_time: str
    success: bool
    post_group_id: Optional[str] = None
    error: Optional[str] = None
    retryable: bool = False


class PubloraQueueScheduler:
    """
    Production-ready queue-based scheduler for Publora API

    Features:
    - Platform-specific rate limiting
    - Peak engagement time optimization
    - Exponential backoff retry logic
    - Failed post recovery
    - Comprehensive error handling
    """

    def __init__(self, api_key: str, **options):
        self.api_key = api_key
        self.base_url = options.get('base_url', 'https://api.publora.com/api/v1')
        self.max_retries = options.get('max_retries', 3)
        self.retry_delay = options.get('retry_delay', 1.0)

        # API rate limit tracking
        self.rate_limit_remaining = float('inf')
        self.rate_limit_reset = 0

        # Platform-specific limits (posts per hour / per day)
        self.platform_limits = {
            'twitter':   {'per_hour': 10, 'per_day': 50},
            'linkedin':  {'per_hour': 5,  'per_day': 20},
            'instagram': {'per_hour': 3,  'per_day': 10},
            'threads':   {'per_hour': 10, 'per_day': 50},
            'tiktok':    {'per_hour': 2,  'per_day': 10},
            'facebook':  {'per_hour': 5,  'per_day': 25},
            'bluesky':   {'per_hour': 10, 'per_day': 100},
        }

        # Peak engagement hours (UTC)
        self.peak_hours = {
            'twitter':   [13, 14, 15, 16],
            'linkedin':  [10, 11, 12],
            'instagram': [11, 12, 13, 14, 19, 20, 21],
            'threads':   [12, 13, 14, 15],
            'tiktok':    [15, 16, 17, 18, 19, 20, 21],
            'facebook':  [13, 14, 15, 16],
            'bluesky':   [14, 15, 16, 17],
        }

        # Track scheduled posts per platform
        self.post_counts: Dict[str, Dict[str, int]] = {
            'hourly': defaultdict(int),
            'daily': defaultdict(int)
        }

        # Failed posts queue for retry
        self.failed_queue: List[Dict] = []

        # Results tracking
        self.results: List[ScheduleResult] = []

    def _request(
        self,
        method: str,
        endpoint: str,
        data: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """Make an API request with rate limit handling and retries"""

        # Wait if API rate limited
        self._wait_for_rate_limit()

        headers = {
            'Content-Type': 'application/json',
            'x-publora-key': self.api_key
        }

        for attempt in range(self.max_retries + 1):
            try:
                response = requests.request(
                    method,
                    f'{self.base_url}{endpoint}',
                    headers=headers,
                    json=data,
                    timeout=30
                )

                # Update rate limit tracking
                self._update_rate_limits(response.headers)

                if response.ok:
                    return {'success': True, 'data': response.json()}

                error_data = response.json() if response.text else {}
                message = error_data.get('error') or error_data.get('message') or 'Unknown error'

                # Handle rate limit (429)
                if response.status_code == 429:
                    retry_after = error_data.get('retryAfter', 60)
                    print(f'Rate limited. Waiting {retry_after}s...')
                    time.sleep(retry_after)
                    continue

                # Don't retry client errors (4xx) except 429
                if 400 <= response.status_code < 500:
                    return {
                        'success': False,
                        'error': message,
                        'status': response.status_code,
                        'retryable': False
                    }

                # Retry server errors (5xx)
                if attempt < self.max_retries:
                    delay = self.retry_delay * (2 ** attempt)
                    print(f'Error (attempt {attempt + 1}/{self.max_retries + 1}). Retrying in {delay}s...')
                    time.sleep(delay)
                    continue

                return {
                    'success': False,
                    'error': message,
                    'status': response.status_code,
                    'retryable': True
                }

            except requests.exceptions.RequestException as e:
                if attempt < self.max_retries:
                    delay = self.retry_delay * (2 ** attempt)
                    print(f'Network error. Retrying in {delay}s...')
                    time.sleep(delay)
                    continue

                return {
                    'success': False,
                    'error': str(e),
                    'status': 0,
                    'retryable': True
                }

        return {'success': False, 'error': 'Max retries exceeded', 'retryable': True}

    def _wait_for_rate_limit(self):
        """Wait if we've hit the API rate limit"""
        if self.rate_limit_remaining <= 1:
            wait_time = self.rate_limit_reset - time.time()
            if wait_time > 0:
                print(f'API rate limit reached. Waiting {wait_time:.1f}s...')
                time.sleep(wait_time + 0.1)

    def _update_rate_limits(self, headers: Dict):
        """Update rate limit tracking from response headers"""
        remaining = headers.get('X-RateLimit-Remaining')
        reset = headers.get('X-RateLimit-Reset')

        if remaining is not None:
            self.rate_limit_remaining = int(remaining)
        if reset is not None:
            self.rate_limit_reset = int(reset)

    def _get_platform_name(self, platform_id: str) -> str:
        """Get platform name from platform ID (e.g., 'twitter-123' -> 'twitter')"""
        return platform_id.split('-')[0]

    def _can_post_at(self, platform: str, dt: datetime) -> bool:
        """Check if a platform can accept a post at the given time"""
        limits = self.platform_limits.get(platform)
        if not limits:
            return True

        hour_key = f"{platform}-{dt.strftime('%Y-%m-%dT%H')}"
        day_key = f"{platform}-{dt.strftime('%Y-%m-%d')}"

        hourly_count = self.post_counts['hourly'][hour_key]
        daily_count = self.post_counts['daily'][day_key]

        return hourly_count < limits['per_hour'] and daily_count < limits['per_day']

    def _record_post(self, platform: str, dt: datetime):
        """Record a scheduled post for rate limiting"""
        hour_key = f"{platform}-{dt.strftime('%Y-%m-%dT%H')}"
        day_key = f"{platform}-{dt.strftime('%Y-%m-%d')}"

        self.post_counts['hourly'][hour_key] += 1
        self.post_counts['daily'][day_key] += 1

    def _is_peak_time(self, platform: str, dt: datetime) -> bool:
        """Check if the given time is a peak engagement hour"""
        hours = self.peak_hours.get(platform)
        return hours is None or dt.hour in hours

    def _find_next_optimal_slot(
        self,
        platforms: List[str],
        after: datetime
    ) -> datetime:
        """Find the next optimal time slot considering all platforms"""
        candidate = after.replace(minute=0, second=0, microsecond=0)

        # Search up to 1 week ahead
        for _ in range(168):
            candidate += timedelta(hours=1)

            # Check if all platforms can post at this time
            all_can_post = all(
                self._can_post_at(self._get_platform_name(p), candidate)
                for p in platforms
            )

            if not all_can_post:
                continue

            # Prefer peak times
            any_peak = any(
                self._is_peak_time(self._get_platform_name(p), candidate)
                for p in platforms
            )

            if any_peak:
                return candidate

        # Fallback: return next hour
        return after + timedelta(hours=1)

    def schedule_batch(
        self,
        posts: List[Dict],
        start_after: Optional[datetime] = None
    ) -> List[ScheduleResult]:
        """Schedule a batch of posts optimally"""

        if start_after is None:
            start_after = datetime.now(timezone.utc)

        results = []
        last_time = start_after

        print(f'\nScheduling {len(posts)} posts...')

        for i, post in enumerate(posts):
            scheduled_time = self._find_next_optimal_slot(post['platforms'], last_time)

            # Record for platform rate limiting
            for p in post['platforms']:
                self._record_post(self._get_platform_name(p), scheduled_time)

            payload = {
                'content': post['content'],
                'platforms': post['platforms'],
                'scheduledTime': scheduled_time.isoformat()
            }

            if 'mediaUrls' in post:
                payload['mediaUrls'] = post['mediaUrls']
            if 'platformSettings' in post:
                payload['platformSettings'] = post['platformSettings']

            response = self._request('POST', '/create-post', payload)

            result = ScheduleResult(
                index=i,
                content=post['content'][:50],
                platforms=post['platforms'],
                scheduled_time=scheduled_time.isoformat(),
                success=response['success']
            )

            if response['success']:
                result.post_group_id = response['data']['postGroupId']
                print(f"[{i + 1}/{len(posts)}] ✓ Scheduled: {result.content}...")
            else:
                result.error = response.get('error')
                result.retryable = response.get('retryable', False)
                print(f"[{i + 1}/{len(posts)}] ✗ Failed: {result.error}")

                if result.retryable:
                    self.failed_queue.append({**post, 'original_index': i})

            results.append(result)
            last_time = scheduled_time

            # Small delay between API calls
            time.sleep(0.1)

        self.results = results
        return results

    def retry_failed(self) -> List[ScheduleResult]:
        """Retry failed posts from the queue"""
        if not self.failed_queue:
            print('No failed posts to retry.')
            return []

        print(f'\nRetrying {len(self.failed_queue)} failed posts...')

        to_retry = self.failed_queue.copy()
        self.failed_queue = []

        return self.schedule_batch(to_retry)

    def check_post_statuses(self, post_group_ids: List[str]) -> List[Dict]:
        """Check status of scheduled posts and handle failures"""
        statuses = []

        for post_group_id in post_group_ids:
            response = self._request('GET', f'/get-post/{post_group_id}')

            if not response['success']:
                statuses.append({
                    'post_group_id': post_group_id,
                    'status': 'error',
                    'error': response.get('error')
                })
                continue

            post_group = response['data']
            status = {
                'post_group_id': post_group_id,
                'status': post_group.get('status'),
                'posts': []
            }

            for post in post_group.get('posts', []):
                post_status = {
                    'platform': post.get('platform'),
                    'platform_id': post.get('platformId'),
                    'status': post.get('status')
                }

                if post.get('status') == 'published':
                    post_status['published_url'] = post.get('publishedUrl')
                elif post.get('status') == 'failed':
                    post_status['error'] = post.get('error')

                status['posts'].append(post_status)

            statuses.append(status)

        return statuses

    def get_summary(self) -> Dict[str, int]:
        """Get summary of scheduling results"""
        successful = sum(1 for r in self.results if r.success)
        failed = sum(1 for r in self.results if not r.success)
        retryable = sum(1 for r in self.results if not r.success and r.retryable)

        return {
            'total': len(self.results),
            'successful': successful,
            'failed': failed,
            'retryable': retryable,
            'failed_queue_size': len(self.failed_queue)
        }


# ============================================
# USAGE EXAMPLE
# ============================================

if __name__ == '__main__':
    scheduler = PubloraQueueScheduler('YOUR_API_KEY')

    # Define posts to schedule
    posts = [
        {
            'content': 'Morning motivation! Start your week with intention. #MondayMotivation',
            'platforms': ['twitter-123456', 'linkedin-ABC123']
        },
        {
            'content': 'Product update: We just shipped a major new feature...',
            'platforms': ['twitter-123456', 'linkedin-ABC123', 'threads-789012']
        },
        {
            'content': 'Behind the scenes of our latest photoshoot',
            'platforms': ['instagram-DEF456', 'tiktok-GHI789']
        },
        {
            'content': 'Weekly tips thread: 5 things I learned this week...',
            'platforms': ['twitter-123456', 'threads-789012']
        },
        {
            'content': 'Customer success story: How Company X grew 300%',
            'platforms': ['linkedin-ABC123']
        }
    ]

    # Schedule all posts
    results = scheduler.schedule_batch(posts)

    # Get summary
    summary = scheduler.get_summary()
    print('\n=== SCHEDULING SUMMARY ===')
    print(f"Total: {summary['total']}")
    print(f"Successful: {summary['successful']}")
    print(f"Failed: {summary['failed']}")
    print(f"Retryable: {summary['retryable']}")

    # Retry any failed posts
    if summary['retryable'] > 0:
        print('\nRetrying failed posts...')
        scheduler.retry_failed()

    # Later: Check publishing status
    successful_ids = [r.post_group_id for r in results if r.success and r.post_group_id]

    if successful_ids:
        print('\n=== POST STATUSES ===')
        # Note: Only check after scheduled time has passed
        statuses = scheduler.check_post_statuses(successful_ids)

        for status in statuses:
            print(f"\nPost {status['post_group_id']}: {status['status']}")

            for post in status.get('posts', []):
                if post['status'] == 'published':
                    print(f"  ✓ {post['platform']}: {post.get('published_url', 'N/A')}")
                elif post['status'] == 'failed':
                    print(f"  ✗ {post['platform']}: {post.get('error', 'Unknown error')}")
                else:
                    print(f"  ○ {post['platform']}: {post['status']}")
```

### Handling Common Edge Cases

| Scenario | Handling Strategy |
|----------|-------------------|
| **API rate limit (429)** | Wait for `retryAfter` seconds, then retry |
| **Server error (5xx)** | Exponential backoff retry (1s, 2s, 4s) |
| **Network timeout** | Retry up to 3 times with backoff |
| **Invalid API key (401)** | Stop immediately, don't retry |
| **Plan limit reached (403)** | Stop immediately, notify user |
| **Invalid content (400)** | Skip post, log error, continue |
| **Partial publish failure** | Poll status, retry failed platforms |
| **Platform rate limit** | Queue and schedule for later time slot |

### Monitoring Published Posts

After scheduling, poll for status to detect and handle failures:

```javascript
async function monitorScheduledPosts(scheduler, postGroupIds, pollInterval = 60000) {
  const pending = new Set(postGroupIds);
  const results = { published: [], failed: [], partial: [] };

  while (pending.size > 0) {
    const statuses = await scheduler.checkPostStatuses([...pending]);

    for (const status of statuses) {
      if (status.status === 'published') {
        results.published.push(status);
        pending.delete(status.postGroupId);
      } else if (status.status === 'failed') {
        results.failed.push(status);
        pending.delete(status.postGroupId);
      } else if (status.status === 'partially_published') {
        results.partial.push(status);
        pending.delete(status.postGroupId);
      }
      // 'scheduled' and 'processing' remain in pending
    }

    if (pending.size > 0) {
      console.log(`Waiting for ${pending.size} posts...`);
      await new Promise(r => setTimeout(r, pollInterval));
    }
  }

  return results;
}
```

## Best Practices

1. **Implement exponential backoff** when rate limited. Don't hammer the API.

2. **Track rate limit headers** to preemptively slow down before hitting limits.

3. **Distribute posts evenly** across the day rather than bulk-scheduling for the same minute.

4. **Use platform-specific optimal times** but always test with your audience.

5. **Monitor analytics** to discover your own peak engagement times.

6. **Cache rate limit state** if making requests from multiple processes.

7. **Handle partial failures** gracefully when posting to multiple platforms.

8. **Queue retryable failures** for automatic retry instead of failing permanently.

9. **Poll post status after scheduled time** to detect platform-level failures that occur during publishing.

10. **Log all API responses** including error details for debugging and monitoring.


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
