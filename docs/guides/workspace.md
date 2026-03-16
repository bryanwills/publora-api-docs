# Workspace / B2B API

> **Note:** Workspace access is not enabled by default. Contact Publora support at **serge@publora.com** to enable Workspace API access for your account.

This guide covers how to use the Publora Workspace API to manage multiple users under your account. This is designed for B2B integrations where you need to create and manage social media posting on behalf of your customers.

## How It Works

The Workspace API lets you create **managed users** under your workspace. Each managed user gets their own isolated environment with their own social media connections and posts. You control everything through your workspace API key.

### Core Concepts

- **Workspace:** Your B2B account that can manage multiple users.
- **Managed User:** A user created under your workspace. They have their own social connections and posts, but you control them via the API. User IDs are MongoDB ObjectIds (24-character hex strings, e.g., `6626a1f5e4b0c91a2d3f4567`). The API returns user IDs as `_id` in response bodies (the MongoDB `_id` field).
- **Connection URL:** A temporary OAuth link you generate and send to a managed user so they can connect their social media accounts.
- **Per-User API Key:** An optional API key generated for a specific managed user, allowing direct API access scoped to that user.

### Authentication Headers

| Header | Purpose |
|---|---|
| `x-publora-key` | Your workspace API key (required on all requests) |
| `x-publora-user-id` | The managed user's ID (required when acting on behalf of a user) |

### Rate Limits

Each managed user has a `dailyPostsLeft` field (default: 100) returned in the user object. Note: this field is stored but **not currently enforced** as a hard limit. The actual posting limits are determined by the workspace owner's plan entitlements: `monthlyPosts` (total posts per billing cycle), `scheduledPosts` (maximum concurrent scheduled posts), and `scheduleHorizonDays` (how far in advance posts can be scheduled). These are enforced by the limits service at scheduling time.

### Endpoints Overview

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/workspace/users` | List all managed users |
| `POST` | `/api/v1/workspace/users` | Create a new managed user |
| `DELETE` | `/api/v1/workspace/users/:userId` | Detach a managed user (preserves user record, removes workspace association) |
| `POST` | `/api/v1/workspace/users/:userId/api-key` | Generate a per-user API key |
| `POST` | `/api/v1/workspace/users/:userId/connection-url` | Generate an OAuth connection link |

## Examples

### Create a Managed User

> **Note:** This endpoint returns HTTP **201 Created** on success, not 200.

**JavaScript (fetch)**

```javascript
const response = await fetch(
  'https://api.publora.com/api/v1/workspace/users',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    },
    body: JSON.stringify({
      username: 'client@example.com',
      displayName: 'Acme Corp'
    })
  }
);

const { user } = await response.json();
console.log('Managed user created:', user._id);
console.log('Username:', user.username);
console.log('Display name:', user.displayName);
```

**Python (requests)**

```python
import requests

response = requests.post(
    'https://api.publora.com/api/v1/workspace/users',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    },
    json={
        'username': 'client@example.com',
        'displayName': 'Acme Corp'
    }
)

data = response.json()
user = data['user']
print(f"Managed user created: {user['_id']}")
print(f"Username: {user['username']}")
print(f"Display name: {user['displayName']}")
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/workspace/users \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -d '{
    "username": "client@example.com",
    "displayName": "Acme Corp"
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const { data } = await axios.post(
  'https://api.publora.com/api/v1/workspace/users',
  {
    username: 'client@example.com',
    displayName: 'Acme Corp'
  },
  {
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    }
  }
);

const user = data.user;
console.log('Managed user created:', user._id);
console.log('Username:', user.username);
console.log('Display name:', user.displayName);
```

---

### List All Managed Users

Each user object in the response includes the following fields:

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | The user's MongoDB ObjectId |
| `displayName` | `string` | The user's display name |
| `username` | `string` | The user's email/username |
| `role` | `string` | The user's role in the workspace |
| `dailyPostsLeft` | `number` | Remaining daily post quota |
| `connectionsPageUrl` | `string` | The user's connections page URL (if generated) |
| `connectionsPageTokenExpiresAt` | `string` | ISO 8601 expiry timestamp for the connections page token |

**JavaScript (fetch)**

```javascript
const response = await fetch(
  'https://api.publora.com/api/v1/workspace/users',
  {
    headers: { 'x-publora-key': 'YOUR_API_KEY' }
  }
);

const { users } = await response.json();

console.log(`Total managed users: ${users.length}`);
for (const user of users) {
  console.log(`  - ${user._id}: ${user.displayName} (${user.username}), role: ${user.role}, dailyPostsLeft: ${user.dailyPostsLeft}`);
}
```

**Python (requests)**

```python
import requests

response = requests.get(
    'https://api.publora.com/api/v1/workspace/users',
    headers={'x-publora-key': 'YOUR_API_KEY'}
)

data = response.json()
users = data['users']

print(f"Total managed users: {len(users)}")
for user in users:
    print(f"  - {user['_id']}: {user['displayName']} ({user['username']})")
```

**cURL**

```bash
curl -s https://api.publora.com/api/v1/workspace/users \
  -H "x-publora-key: YOUR_API_KEY" | jq '.users[] | "\(._id): \(.displayName) (\(.username))"'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const { data } = await axios.get(
  'https://api.publora.com/api/v1/workspace/users',
  {
    headers: { 'x-publora-key': 'YOUR_API_KEY' }
  }
);

const users = data.users;
console.log(`Total managed users: ${users.length}`);
for (const user of users) {
  console.log(`  - ${user._id}: ${user.displayName} (${user.username})`);
}
```

---

### Generate a Connection URL

A connection URL is a temporary OAuth link that you send to your managed user. When they open it, they can authorize their social media accounts (Twitter, LinkedIn, Instagram, etc.) to be managed through your workspace.

**Optional body parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `expiresInDays` | `number` | `90` | Number of days until the connection URL expires. Must be a positive number. No upper bound is enforced by the API. |
| `rotate` | `boolean` | `false` | If `true`, invalidates any previously generated connection URL for this user and generates a fresh one |

**JavaScript (fetch)**

```javascript
const userId = '6626a1f5e4b0c91a2d3f4567'; // The managed user's ID

const response = await fetch(
  `https://api.publora.com/api/v1/workspace/users/${userId}/connection-url`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    },
    body: JSON.stringify({
      expiresInDays: 30,  // Optional: default is 90
      rotate: true         // Optional: invalidate previous URL
    })
  }
);

const data = await response.json();
console.log('Connection URL:', data.url);
console.log('Expires:', data.tokenExpiresAt); // Default TTL: 90 days

// Send this URL to your user via email, in-app notification, etc.
// When they open it, they will be guided through connecting their social accounts.
```

**Python (requests)**

```python
import requests

user_id = '6626a1f5e4b0c91a2d3f4567'

response = requests.post(
    f'https://api.publora.com/api/v1/workspace/users/{user_id}/connection-url',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    }
)

data = response.json()
print(f"Connection URL: {data['url']}")
print(f"Expires: {data['tokenExpiresAt']}")  # Default TTL: 90 days

# Send this URL to your user via email, in-app notification, etc.
```

**cURL**

```bash
curl -X POST "https://api.publora.com/api/v1/workspace/users/6626a1f5e4b0c91a2d3f4567/connection-url" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY"
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const userId = '6626a1f5e4b0c91a2d3f4567';

const { data } = await axios.post(
  `https://api.publora.com/api/v1/workspace/users/${userId}/connection-url`,
  {},
  {
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    }
  }
);

console.log('Connection URL:', data.url);
console.log('Expires:', data.tokenExpiresAt); // Default TTL: 90 days

// Send this URL to your user via email, in-app notification, etc.
```

---

### Generate a Per-User API Key

If you want a managed user to have their own API key (scoped only to their data), you can generate one.

> **Requirement:** The workspace **owner's** plan must include the `apiAccess` entitlement. The `apiAccess` check is performed against the workspace owner's subscription, not the managed user's plan. If the owner's plan does not include API access, this endpoint will return a `403` error. Contact Publora support to ensure the appropriate entitlements are configured.

**JavaScript (fetch)**

```javascript
const userId = '6626a1f5e4b0c91a2d3f4567';

const response = await fetch(
  `https://api.publora.com/api/v1/workspace/users/${userId}/api-key`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    }
  }
);

const data = await response.json();
console.log('Per-user API key:', data.apiKey);
console.log('User ID:', data.userId);
console.log('Message:', data.message);

// This key can be used in the x-publora-key header
// and will only have access to this specific user's data.
```

**Python (requests)**

```python
import requests

user_id = '6626a1f5e4b0c91a2d3f4567'

response = requests.post(
    f'https://api.publora.com/api/v1/workspace/users/{user_id}/api-key',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    }
)

data = response.json()
print(f"Per-user API key: {data['apiKey']}")
print(f"User ID: {data['userId']}")
print(f"Message: {data['message']}")
```

**cURL**

```bash
curl -X POST "https://api.publora.com/api/v1/workspace/users/6626a1f5e4b0c91a2d3f4567/api-key" \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY"
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const userId = '6626a1f5e4b0c91a2d3f4567';

const { data } = await axios.post(
  `https://api.publora.com/api/v1/workspace/users/${userId}/api-key`,
  {},
  {
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY'
    }
  }
);

console.log('Per-user API key:', data.apiKey);
console.log('User ID:', data.userId);
console.log('Message:', data.message);
```

---

### Post on Behalf of a Managed User

To create posts for a managed user, include the `x-publora-user-id` header with the managed user's ID alongside your workspace API key.

**JavaScript (fetch)**

```javascript
const userId = '6626a1f5e4b0c91a2d3f4567';

const response = await fetch('https://api.publora.com/api/v1/create-post', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY',
    'x-publora-user-id': userId
  },
  body: JSON.stringify({
    content: 'Exciting update from Acme Corp! We just hit 10,000 customers.',
    platforms: ['twitter-123456', 'linkedin-ABCDEF'],
    scheduledTime: '2026-03-15T14:00:00.000Z'
  })
});

const post = await response.json();
console.log(`Post created for user ${userId}:`, post.postGroupId);
```

**Python (requests)**

```python
import requests

user_id = '6626a1f5e4b0c91a2d3f4567'

response = requests.post(
    'https://api.publora.com/api/v1/create-post',
    headers={
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY',
        'x-publora-user-id': user_id
    },
    json={
        'content': 'Exciting update from Acme Corp! We just hit 10,000 customers.',
        'platforms': ['twitter-123456', 'linkedin-ABCDEF'],
        'scheduledTime': '2026-03-15T14:00:00.000Z'
    }
)

post = response.json()
print(f"Post created for user {user_id}: {post['postGroupId']}")
```

**cURL**

```bash
curl -X POST https://api.publora.com/api/v1/create-post \
  -H "Content-Type: application/json" \
  -H "x-publora-key: YOUR_API_KEY" \
  -H "x-publora-user-id: 6626a1f5e4b0c91a2d3f4567" \
  -d '{
    "content": "Exciting update from Acme Corp! We just hit 10,000 customers.",
    "platforms": ["twitter-123456", "linkedin-ABCDEF"],
    "scheduledTime": "2026-03-15T14:00:00.000Z"
  }'
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const userId = '6626a1f5e4b0c91a2d3f4567';

const { data: post } = await axios.post(
  'https://api.publora.com/api/v1/create-post',
  {
    content: 'Exciting update from Acme Corp! We just hit 10,000 customers.',
    platforms: ['twitter-123456', 'linkedin-ABCDEF'],
    scheduledTime: '2026-03-15T14:00:00.000Z'
  },
  {
    headers: {
      'Content-Type': 'application/json',
      'x-publora-key': 'YOUR_API_KEY',
      'x-publora-user-id': userId
    }
  }
);

console.log(`Post created for user ${userId}:`, post.postGroupId);
```

---

### Detach a Managed User

On success, the response body is `{ "success": true }`.

**JavaScript (fetch)**

```javascript
const userId = '6626a1f5e4b0c91a2d3f4567';

const response = await fetch(
  `https://api.publora.com/api/v1/workspace/users/${userId}`,
  {
    method: 'DELETE',
    headers: { 'x-publora-key': 'YOUR_API_KEY' }
  }
);

if (response.ok) {
  const data = await response.json();
  console.log(`User ${userId} has been detached. Success:`, data.success);
} else {
  const error = await response.json();
  console.error('Failed to detach user:', error.error || error.message);
}
```

**Python (requests)**

```python
import requests

user_id = '6626a1f5e4b0c91a2d3f4567'

response = requests.delete(
    f'https://api.publora.com/api/v1/workspace/users/{user_id}',
    headers={'x-publora-key': 'YOUR_API_KEY'}
)

if response.ok:
    print(f"User {user_id} has been detached.")
else:
    error = response.json()
    print(f"Failed to detach user: {error.get('error') or error.get('message')}")
```

**cURL**

```bash
curl -X DELETE "https://api.publora.com/api/v1/workspace/users/6626a1f5e4b0c91a2d3f4567" \
  -H "x-publora-key: YOUR_API_KEY"
```

**Node.js (axios)**

```javascript
const axios = require('axios');

const userId = '6626a1f5e4b0c91a2d3f4567';

try {
  await axios.delete(
    `https://api.publora.com/api/v1/workspace/users/${userId}`,
    {
      headers: { 'x-publora-key': 'YOUR_API_KEY' }
    }
  );

  console.log(`User ${userId} has been detached.`);
} catch (error) {
  console.error('Failed to detach user:', error.response?.data?.error);
}
```

---

### Full B2B Onboarding Workflow

This example shows the complete workflow: create a managed user, generate a connection URL, wait for them to connect, then post on their behalf.

**JavaScript (fetch)**

```javascript
async function onboardClient(username, displayName) {
  const headers = {
    'Content-Type': 'application/json',
    'x-publora-key': 'YOUR_API_KEY'
  };

  // Step 1: Create the managed user
  const createResponse = await fetch(
    'https://api.publora.com/api/v1/workspace/users',
    {
      method: 'POST',
      headers,
      body: JSON.stringify({ username, displayName })
    }
  );

  const { user } = await createResponse.json();
  console.log(`1. Created user: ${user._id} (${user.displayName})`);

  // Step 2: Generate a connection URL for them to auth their social accounts
  const connectionResponse = await fetch(
    `https://api.publora.com/api/v1/workspace/users/${user._id}/connection-url`,
    {
      method: 'POST',
      headers
    }
  );

  const connectionData = await connectionResponse.json();
  console.log(`2. Connection URL: ${connectionData.url}`);
  console.log(`   Expires: ${connectionData.tokenExpiresAt}`);
  console.log('   Send this link to the client so they can connect their social accounts.');

  // Step 3: (After user connects accounts) Check their connections
  // In production, you would wait for a webhook or poll periodically.
  const connectionsResponse = await fetch(
    'https://api.publora.com/api/v1/platform-connections',
    {
      headers: {
        'x-publora-key': 'YOUR_API_KEY',
        'x-publora-user-id': user._id
      }
    }
  );

  const { connections } = await connectionsResponse.json();
  console.log(`3. User has ${connections.length} connected accounts.`);

  if (connections.length === 0) {
    console.log('   User has not connected any accounts yet.');
    return user;
  }

  // Step 4: Post on their behalf
  const platforms = connections.map(c => c.platformId);

  const postResponse = await fetch('https://api.publora.com/api/v1/create-post', {
    method: 'POST',
    headers: {
      ...headers,
      'x-publora-user-id': user._id
    },
    body: JSON.stringify({
      content: `Welcome to ${displayName}'s social presence, powered by our platform!`,
      platforms,
      scheduledTime: '2026-03-15T14:00:00.000Z'
    })
  });

  const post = await postResponse.json();
  console.log(`4. Post scheduled for user ${user._id}: ${post.postGroupId}`);

  return user;
}

// Usage
const client = await onboardClient('acme-corp', 'Acme Corp');
```

**Python (requests)**

```python
import requests


def onboard_client(username, display_name):
    api_url = 'https://api.publora.com/api/v1'
    headers = {
        'Content-Type': 'application/json',
        'x-publora-key': 'YOUR_API_KEY'
    }

    # Step 1: Create the managed user
    user_response = requests.post(
        f'{api_url}/workspace/users',
        headers=headers,
        json={'username': username, 'displayName': display_name}
    )
    user = user_response.json()['user']
    print(f"1. Created user: {user['_id']} ({user['displayName']})")

    # Step 2: Generate a connection URL
    connection_response = requests.post(
        f"{api_url}/workspace/users/{user['_id']}/connection-url",
        headers=headers
    )
    connection_data = connection_response.json()
    print(f"2. Connection URL: {connection_data['url']}")
    print(f"   Expires: {connection_data['tokenExpiresAt']}")
    print('   Send this link to the client.')

    # Step 3: Check their connections (after they connect)
    user_headers = {**headers, 'x-publora-user-id': user['_id']}
    connections_response = requests.get(
        f'{api_url}/platform-connections',
        headers=user_headers
    )
    connections = connections_response.json()['connections']
    print(f"3. User has {len(connections)} connected accounts.")

    if not connections:
        print('   User has not connected any accounts yet.')
        return user

    # Step 4: Post on their behalf
    platform_ids = [c['platformId'] for c in connections]
    post_response = requests.post(
        f'{api_url}/create-post',
        headers=user_headers,
        json={
            'content': f"Welcome to {display_name}'s social presence, powered by our platform!",
            'platforms': platform_ids,
            'scheduledTime': '2026-03-15T14:00:00.000Z'
        }
    )
    post = post_response.json()
    print(f"4. Post scheduled for user {user['_id']}: {post['postGroupId']}")

    return user


# Usage
client = onboard_client('acme-corp', 'Acme Corp')
```

## Best Practices

1. **Store managed user IDs.** After creating a managed user, persist their `id` in your own database. You will need it for all subsequent operations on their behalf.

2. **Send connection URLs promptly.** Connection URLs expire after 90 days by default (see `tokenExpiresAt` in the response). Send them to your users as soon as they are generated, and regenerate if needed.

3. **Use per-user API keys for client-side integrations.** If your managed users need to interact with the API directly (e.g., from their own dashboard), generate per-user API keys instead of sharing your workspace key.

4. **Never expose your workspace API key.** Your workspace key has full access to all managed users. Keep it server-side only. Use per-user API keys for any client-facing scenarios.

5. **Monitor plan entitlements.** The actual posting limits come from the workspace owner's plan (`monthlyPosts`, `scheduledPosts`, `scheduleHorizonDays`). The `dailyPostsLeft` field is stored per user but not currently enforced. If you are building a high-volume integration, track your plan's monthly and scheduled post limits and queue posts accordingly.

6. **Detach unused users.** If a managed user is no longer needed (e.g., they cancel their subscription with you), detach them via the `DELETE` endpoint. This removes the workspace association but preserves the user record.

7. **Always include both headers when acting on behalf of a user.** For any post, connection, or media operation on a managed user's behalf, you need both `x-publora-key` (your workspace key) and `x-publora-user-id` (the managed user's ID).

## Common Issues

| Problem | Cause | Solution |
|---|---|---|
| `409` "User with this email already exists" | A managed user with the same `username` (email) already exists in the workspace | Use a different email or retrieve the existing user via `GET /api/v1/workspace/users` |
| `403` "Workspace access is not enabled for this key" | Workspace feature not enabled for your account | Contact serge@publora.com to enable Workspace API access |
| `401` when using workspace endpoints | Invalid workspace API key | Verify the `x-publora-key` value is your workspace-level API key |
| `403` `"User is not managed by key"` when posting on behalf of a user | The `x-publora-user-id` refers to a user whose `parentUser` is not set to your user ID | Ensure the managed user has `parentUser` set to your user ID (i.e., they belong to your workspace) |
| `404` `"User not found or not managed by this key"` when detaching a user | The user ID does not exist or is not managed by your workspace | Verify the user ID is correct and belongs to your workspace via `GET /api/v1/workspace/users` |
| User has no connections | They have not opened the connection URL yet, or the URL expired | Generate a new connection URL and send it to the user |
| Connection URL expired | The URL has a limited lifespan | Generate a fresh connection URL via `POST /workspace/users/:userId/connection-url` |
| `403` limit reached | Monthly or scheduled post limit exceeded per the workspace owner's plan entitlements | Check your plan's `monthlyPosts` and `scheduledPosts` limits, wait for next billing cycle, or contact Publora to upgrade |
| Posts not appearing for a managed user | Using your own API key without the `x-publora-user-id` header | Include the `x-publora-user-id` header so the post is created under the managed user's account |
| `403` "API access is not enabled for this workspace owner" | The workspace owner's plan does not include the `apiAccess` entitlement, which is required for generating per-user API keys | Contact Publora support to ensure the workspace owner's plan includes API access |
| Per-user API key does not work | Key was regenerated, invalidating the old one | Use the latest generated key; generating a new key invalidates previous keys |


---

*[Publora](https://publora.com) is built by [Creative Content Crafts, Inc.](https://cccrafts.ai) Need AI-powered content creation for LinkedIn, Threads, and X? Try [Co.Actor](https://co.actor) -- the best AI service for authentic thought leadership at scale.*
