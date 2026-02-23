# Webhooks

Receive real-time notifications when posts are published, fail, or when tokens are expiring.

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/webhooks` | List all webhooks |
| POST | `/webhooks` | Create a webhook |
| PATCH | `/webhooks/:id` | Update a webhook |
| DELETE | `/webhooks/:id` | Delete a webhook |
| POST | `/webhooks/:id/regenerate-secret` | Regenerate signing secret |

## Headers

| Header | Required | Description |
|--------|----------|-------------|
| `x-publora-key` | Yes | Your API key |
| `x-publora-user-id` | No | Managed user ID (workspace only) |

---

## List Webhooks

```
GET https://api.publora.com/api/v1/webhooks
```

### Response

```json
{
  "success": true,
  "webhooks": [
    {
      "_id": "65f8a1b2c3d4e5f6a7b8c9d0",
      "name": "Production Notifications",
      "url": "https://your-app.com/webhooks/publora",
      "events": ["post.published", "post.failed"],
      "isActive": true,
      "failureCount": 0,
      "lastTriggeredAt": "2026-02-22T14:30:00.000Z",
      "createdAt": "2026-02-20T10:00:00.000Z"
    }
  ]
}
```

---

## Create Webhook

```
POST https://api.publora.com/api/v1/webhooks
```

### Request Body

```json
{
  "name": "Production Notifications",
  "url": "https://your-app.com/webhooks/publora",
  "events": ["post.published", "post.failed", "token.expiring"]
}
```

### Response

```json
{
  "success": true,
  "webhook": {
    "_id": "65f8a1b2c3d4e5f6a7b8c9d0",
    "name": "Production Notifications",
    "url": "https://your-app.com/webhooks/publora",
    "events": ["post.published", "post.failed", "token.expiring"],
    "secret": "a1b2c3d4e5f6...your-signing-secret...x9y0z1",
    "isActive": true,
    "createdAt": "2026-02-22T10:00:00.000Z"
  }
}
```

> **Important:** The `secret` is only returned once when creating the webhook. Store it securely for signature verification.

### Available Events

| Event | Description |
|-------|-------------|
| `post.scheduled` | Post was scheduled |
| `post.published` | Post was successfully published |
| `post.failed` | Post failed to publish |
| `token.expiring` | Platform token is expiring soon |

---

## Update Webhook

```
PATCH https://api.publora.com/api/v1/webhooks/:id
```

### Request Body

```json
{
  "name": "Updated Name",
  "url": "https://new-url.com/webhook",
  "events": ["post.failed"],
  "isActive": false
}
```

All fields are optional. Only provided fields will be updated.

---

## Delete Webhook

```
DELETE https://api.publora.com/api/v1/webhooks/:id
```

### Response

```json
{
  "success": true
}
```

---

## Regenerate Secret

```
POST https://api.publora.com/api/v1/webhooks/:id/regenerate-secret
```

### Response

```json
{
  "success": true,
  "secret": "new-secret-here..."
}
```

---

## Webhook Payload

When an event occurs, Publora sends a POST request to your webhook URL:

### Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Publora-Signature` | HMAC-SHA256 signature of the payload |
| `X-Publora-Event` | Event type (e.g., `post.published`) |

### Payload Structure

```json
{
  "event": "post.published",
  "timestamp": "2026-02-22T14:30:00.000Z",
  "data": {
    "postId": "507f1f77bcf86cd799439012",
    "postGroupId": "507f1f77bcf86cd799439011",
    "platform": "linkedin",
    "publishedAt": "2026-02-22T14:30:00.000Z"
  }
}
```

### Event-Specific Data

#### post.published

```json
{
  "postId": "507f1f77bcf86cd799439012",
  "postGroupId": "507f1f77bcf86cd799439011",
  "platform": "linkedin",
  "publishedAt": "2026-02-22T14:30:00.000Z"
}
```

#### post.failed

```json
{
  "postId": "507f1f77bcf86cd799439012",
  "postGroupId": "507f1f77bcf86cd799439011",
  "platform": "threads",
  "error": {
    "code": "PLATFORM_AUTH_EXPIRED",
    "message": "Token expired"
  },
  "failedAt": "2026-02-22T14:30:00.000Z"
}
```

#### token.expiring

```json
{
  "platform": "instagram",
  "platformId": "instagram-17841412345678",
  "username": "yourinstagram",
  "expiresAt": "2026-02-25T08:00:00.000Z"
}
```

---

## Signature Verification

Verify webhook authenticity using HMAC-SHA256:

### Node.js

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}

// Express middleware
app.post('/webhooks/publora', express.json(), (req, res) => {
  const signature = req.headers['x-publora-signature'];
  const event = req.headers['x-publora-event'];

  if (!verifyWebhookSignature(req.body, signature, process.env.PUBLORA_WEBHOOK_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  console.log(`Received ${event}:`, req.body.data);

  // Handle the event
  switch (event) {
    case 'post.published':
      // Update your database, send notification, etc.
      break;
    case 'post.failed':
      // Alert your team, retry logic, etc.
      break;
    case 'token.expiring':
      // Notify user to reconnect
      break;
  }

  res.status(200).json({ received: true });
});
```

### Python (Flask)

```python
import hmac
import hashlib
import json
from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ['PUBLORA_WEBHOOK_SECRET']

def verify_signature(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        json.dumps(payload).encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

@app.route('/webhooks/publora', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Publora-Signature')
    event = request.headers.get('X-Publora-Event')
    payload = request.json

    if not verify_signature(payload, signature, WEBHOOK_SECRET):
        return jsonify({'error': 'Invalid signature'}), 401

    print(f"Received {event}: {payload['data']}")

    if event == 'post.failed':
        # Alert team about failed post
        send_slack_alert(payload['data'])

    return jsonify({'received': True}), 200
```

---

## Webhook Reliability

- Webhooks timeout after 10 seconds
- Failed webhooks are retried (up to 5 consecutive failures)
- After 5 consecutive failures, the webhook is automatically disabled
- Re-enable a disabled webhook by updating `isActive: true`

## Limits

- Maximum 10 webhooks per user
- URL must be HTTPS (HTTP allowed only for localhost during development)
- Internal IPs and cloud metadata endpoints are blocked

---

## Examples

### Create Webhook with cURL

```bash
curl -X POST https://api.publora.com/api/v1/webhooks \
  -H "x-publora-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Webhook",
    "url": "https://my-app.com/webhooks/publora",
    "events": ["post.published", "post.failed"]
  }'
```

### List Webhooks

```bash
curl https://api.publora.com/api/v1/webhooks \
  -H "x-publora-key: YOUR_API_KEY"
```

### Delete Webhook

```bash
curl -X DELETE https://api.publora.com/api/v1/webhooks/65f8a1b2c3d4e5f6a7b8c9d0 \
  -H "x-publora-key: YOUR_API_KEY"
```

---

## Errors

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `"Name, URL, and at least one event are required"` | Missing required fields |
| 400 | `"Invalid URL format"` | Malformed URL |
| 400 | `"Private IP addresses are not allowed"` | URL points to internal network |
| 400 | `"Maximum of 10 webhooks per user"` | Webhook limit reached |
| 401 | `"Invalid API key"` | Bad or missing `x-publora-key` |
| 404 | `"Webhook not found"` | Invalid webhook ID |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) — the best AI service for authentic thought leadership at scale.*
