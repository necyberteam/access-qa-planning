# JSM "My Tickets" API Specification

## Overview

Add an endpoint to the ACCESS JSM Netlify proxy (`access-jsm-api.netlify.app`) that allows users to retrieve their support tickets from Jira Service Management.

## Background

The current proxy supports ticket **creation** only:
- `POST /api/v1/tickets` - Create support tickets
- `POST /api/v1/security-incidents` - Report security incidents

We need to add ticket **retrieval** to enable the ACCESS agent to show users their existing tickets.

## Proposed Endpoint

### `GET /api/v1/my-requests`

Retrieve tickets owned by or participated in by a specific user.

#### Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-User-Email` | Yes | Email address of the user (from Drupal) |
| `X-Acting-User` | Yes | ACCESS ID of the user (from Drupal) |

Both headers are required for dual verification - see [User Identity & Privacy](./jsm-mcp-server-plan.md#user-identity--privacy). These values are extracted server-side by the agent from a validated JWT cookie — they are never supplied by the client. See [Threat Model](./jsm-mcp-server-plan.md#threat-model) for security analysis.

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | string | `open` | Filter by status: `open`, `closed`, `all` |
| `limit` | number | `20` | Maximum tickets to return (max 50) |
| `start` | number | `0` | Pagination offset |

#### Response

```json
{
  "success": true,
  "data": {
    "total": 5,
    "start": 0,
    "limit": 20,
    "requests": [
      {
        "issueKey": "ACCESS-12345",
        "issueUrl": "https://access-ci.atlassian.net/servicedesk/customer/portal/2/ACCESS-12345",
        "summary": "Cannot allocate GPUs on Delta",
        "status": "Waiting for support",
        "statusCategory": "open",
        "createdDate": "2024-01-15T10:30:00.000Z",
        "updatedDate": "2024-01-16T14:22:00.000Z",
        "requestType": "General Support",
        "priority": "Medium"
      }
    ]
  }
}
```

#### Error Response

```json
{
  "success": false,
  "error": "User not found or no access"
}
```

## Atlassian JSM API Reference

The proxy should call the JSM Customer Request API:

### Get Customer Requests

```
GET /rest/servicedeskapi/request
```

**Query Parameters:**
- `requestOwnership=OWNED_REQUESTS` - Tickets created by the user
- `requestOwnership=PARTICIPATED_REQUESTS` - Tickets user participated in
- `requestOwnership=ALL_REQUESTS` - Both (requires permissions)
- `requestStatus=OPEN_REQUESTS` or `CLOSED_REQUESTS` or `ALL_REQUESTS`
- `limit` - Page size
- `start` - Pagination offset

**Authentication:**
The proxy uses a service account with JQL search.

### Recommended Approach: Dual Verification

For security, query must match **both** ACCESS ID and email. This prevents users from viewing other users' tickets even if they know their email.

```
GET /rest/api/3/search
```

With JQL (both conditions required):
```
project = ACCESS
  AND "ACCESS ID" = "jsmith@access-ci.org"
  AND reporter = "john.smith@university.edu"
ORDER BY created DESC
```

**Note**: Tickets without the ACCESS ID field populated will not be returned. This is intentional - we cannot verify ownership of legacy tickets that lack ACCESS ID.

## Implementation Notes

### 1. Netlify Function Structure

Create a new function at `netlify/functions/my-requests.js`:

```javascript
const fetch = require('node-fetch');

exports.handler = async (event) => {
  // Only allow GET
  if (event.httpMethod !== 'GET') {
    return { statusCode: 405, body: JSON.stringify({ error: 'Method not allowed' }) };
  }

  const userEmail = event.headers['x-user-email'];
  if (!userEmail) {
    return { statusCode: 400, body: JSON.stringify({ error: 'X-User-Email header required' }) };
  }

  const params = event.queryStringParameters || {};
  const status = params.status || 'open';
  const limit = Math.min(parseInt(params.limit) || 20, 50);
  const start = parseInt(params.start) || 0;

  // Build JQL
  let statusFilter = '';
  if (status === 'open') {
    statusFilter = 'AND statusCategory != Done';
  } else if (status === 'closed') {
    statusFilter = 'AND statusCategory = Done';
  }

  const jql = `project = ACCESS AND reporter = "${userEmail}" ${statusFilter} ORDER BY created DESC`;

  // Call Atlassian API
  const response = await fetch(
    `${process.env.JIRA_BASE_URL}/rest/api/3/search?jql=${encodeURIComponent(jql)}&maxResults=${limit}&startAt=${start}`,
    {
      headers: {
        'Authorization': `Basic ${Buffer.from(`${process.env.JIRA_EMAIL}:${process.env.JIRA_API_TOKEN}`).toString('base64')}`,
        'Content-Type': 'application/json',
      },
    }
  );

  // Transform and return
  // ...
};
```

### 2. Response Transformation

Map Atlassian fields to our simplified format:
- `key` → `issueKey`
- `fields.summary` → `summary`
- `fields.status.name` → `status`
- `fields.status.statusCategory.key` → `statusCategory` (map to open/closed)
- `fields.created` → `createdDate`
- `fields.updated` → `updatedDate`
- `fields.priority.name` → `priority`

### 3. Security Considerations

- **JWT cookie auth required** — ticket reading must not be enabled until the cookie authentication flow is implemented (see [Prerequisites](./jsm-mcp-server-plan.md#prerequisites))
- Validate email format to prevent JQL injection
- Rate limit requests per user
- Log access for audit trail
- Consider caching with short TTL (1-2 minutes)

## MCP Server Tool

Once the proxy endpoint is ready, add this tool to the JSM MCP server:

```typescript
{
  name: "get_my_tickets",
  description: "Get the user's support tickets from ACCESS. Shows open tickets by default.",
  inputSchema: {
    type: "object",
    properties: {
      user_email: {
        type: "string",
        description: "User's email address"
      },
      status: {
        type: "string",
        enum: ["open", "closed", "all"],
        default: "open",
        description: "Filter by ticket status"
      },
      limit: {
        type: "number",
        default: 10,
        description: "Maximum tickets to return"
      }
    },
    required: ["user_email"]
  }
}
```

## Testing

1. Create a test ticket via the existing endpoint
2. Query using the new endpoint with the same email
3. Verify the created ticket appears in results
4. Test status filters (open/closed/all)
5. Test pagination with limit/start
6. Test error cases (invalid email, no tickets)

## Timeline Estimate

- Proxy endpoint: 2-4 hours
- MCP tool addition: 1 hour
- Testing: 1-2 hours
- Documentation: 30 min

## Design Decisions

1. **Include comments/history or just metadata?**
   - **Decision: Metadata only for list endpoint.** Keep it fast and lightweight. Comments/history available via separate details endpoint.

2. **Support searching by ticket number?**
   - **Decision: Yes.** Add optional `ticket_key` parameter to the list endpoint. Common use case: "What's the status of ACCESS-12345?"

3. **Separate ticket details endpoint?**
   - **Decision: Yes.** Add `GET /api/v1/requests/:key` for full ticket info including comments.

## Complete API Surface

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/my-requests` | List my tickets (metadata only) |
| `GET /api/v1/my-requests?ticket_key=ACCESS-123` | Get specific ticket by key (metadata) |
| `GET /api/v1/requests/:key` | Get full ticket details (description, comments, history) |

---

## Ticket Details Endpoint

### `GET /api/v1/requests/:key`

Retrieve full details for a specific ticket including comments and history.

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `key` | Ticket key (e.g., `ACCESS-12345`) |

#### Response

```json
{
  "success": true,
  "data": {
    "issueKey": "ACCESS-12345",
    "issueUrl": "https://access-ci.atlassian.net/servicedesk/customer/portal/2/ACCESS-12345",
    "summary": "Cannot allocate GPUs on Delta",
    "description": "Full description text here...",
    "status": "Waiting for support",
    "statusCategory": "open",
    "createdDate": "2024-01-15T10:30:00.000Z",
    "updatedDate": "2024-01-16T14:22:00.000Z",
    "requestType": "General Support",
    "priority": "Medium",
    "reporter": {
      "email": "user@university.edu",
      "displayName": "John Smith"
    },
    "comments": [
      {
        "id": "10001",
        "author": {
          "displayName": "Support Agent",
          "isStaff": true
        },
        "body": "Thanks for reporting this. We're looking into it.",
        "created": "2024-01-15T14:00:00.000Z"
      },
      {
        "id": "10002",
        "author": {
          "displayName": "John Smith",
          "isStaff": false
        },
        "body": "Any update on this?",
        "created": "2024-01-16T09:00:00.000Z"
      }
    ],
    "attachments": [
      {
        "filename": "error_screenshot.png",
        "size": 45678,
        "created": "2024-01-15T10:30:00.000Z"
      }
    ]
  }
}
```

#### Error Response

```json
{
  "success": false,
  "error": "Ticket not found"
}
```

### Atlassian API for Details

```
GET /rest/api/3/issue/{issueKey}?expand=comments,changelog
```

Or for Service Desk specifically:
```
GET /rest/servicedeskapi/request/{issueKey}
GET /rest/servicedeskapi/request/{issueKey}/comment
```

---

## Updated MCP Server Tools

Once the proxy endpoints are ready, add these tools to the JSM MCP server:

### `get_my_tickets`

```typescript
{
  name: "get_my_tickets",
  description: "Get the user's support tickets from ACCESS. Shows open tickets by default.",
  inputSchema: {
    type: "object",
    properties: {
      user_email: {
        type: "string",
        description: "User's email address"
      },
      ticket_key: {
        type: "string",
        description: "Specific ticket key to look up (e.g., ACCESS-12345)"
      },
      status: {
        type: "string",
        enum: ["open", "closed", "all"],
        default: "open",
        description: "Filter by ticket status"
      },
      limit: {
        type: "number",
        default: 10,
        description: "Maximum tickets to return"
      }
    },
    required: ["user_email"]
  }
}
```

### `get_ticket_details`

```typescript
{
  name: "get_ticket_details",
  description: "Get full details of a specific ticket including comments and history.",
  inputSchema: {
    type: "object",
    properties: {
      ticket_key: {
        type: "string",
        description: "Ticket key (e.g., ACCESS-12345)"
      }
    },
    required: ["ticket_key"]
  }
}
```
