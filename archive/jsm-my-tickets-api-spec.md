# JSM "My Tickets" API Specification

## Overview

Add endpoints to the ACCESS JSM Netlify proxy that allow users to retrieve their support tickets from Jira Service Management, and corresponding MCP tools so AI assistants can show users their tickets.

## Status

**Pending approval.** OAuth authentication (CILogon) is implemented. The ticket ownership verification approach needs stakeholder review — see [Security: Ticket Ownership](#security-ticket-ownership) for details.

## Background

The current proxy supports ticket **creation** only:
- `POST /api/v1/tickets` - Create support tickets
- `POST /api/v1/security-incidents` - Report security incidents

We need to add ticket **retrieval** to enable the ACCESS agent and MCP-connected AI assistants (Claude, etc.) to show users their existing tickets.

## Identity Model

With OAuth authentication via CILogon now implemented on the MCP servers, user identity comes from two trusted sources:

1. **OAuth (Claude Connectors):** User authenticates via CILogon. Their ACCESS ID (`username@access-ci.org`) is verified by the MCP server's Bearer token middleware and stored in `RequestContext.actingUser`.

2. **Agent (REST API):** User's JWT cookie is validated by the agent. The ACCESS ID is forwarded to the MCP server via the `X-Acting-User` header.

**Key principle:** User identity is NEVER supplied as a tool parameter. The `get_my_tickets` tool has no `user_email` or `access_id` parameter — identity comes entirely from the authentication context.

## Security: Ticket Ownership

### The Problem

We need to guarantee that users can only see their own tickets. This is harder than it sounds:

- **`customfield_10103` (ACCESS ID):** User-editable form field. Anyone submitting a ticket through the JSM portal can type another person's ACCESS ID. Cannot be trusted for ownership.

- **`reporter` field:** Set by JSM based on the submitter, but tickets created via the service account without `raiseOnBehalfOf` have the service account as reporter. Also, email-submitted tickets use the from address (spoofable).

- **Atlassian OAuth (3LO):** Would let users authenticate directly with Jira and use `requestOwnership=OWNED_REQUESTS`. However, **JSM customer (portal-only) accounts cannot complete the OAuth consent flow on Jira Cloud.** This is a confirmed Atlassian limitation — unlicensed users are redirected away from the consent screen. Licensing every ACCESS researcher is not feasible.
  - Sources: [Atlassian community: portal-only OAuth](https://community.developer.atlassian.com/t/authenticating-with-the-service-desk-rest-api-as-a-customer-portal-only-user/24978), [Customer REST API auth](https://community.developer.atlassian.com/t/is-it-possible-for-a-customer-to-login-and-create-service-desk-tickets-via-rest-api/14750)

### Proposed Solution: Verified Owner Field

Create a hidden JSM custom field (`Verified Owner`) that:

1. Is **NOT on any customer-facing portal form** — external users cannot set it
2. Is **only populated via the API** by our service account when tickets are created through authenticated channels (MCP server with OAuth, agent with JWT cookie)
3. Contains the user's verified ACCESS ID (e.g., `apasquale@access-ci.org`)
4. Is used as the ownership filter in My Tickets queries

**Security properties:**
- External users cannot set the field (it's not on any form they see)
- Only our authenticated ticket creation paths write to it
- JQL queries filter on this field, not the user-editable ACCESS ID field
- Legacy tickets (created before this field exists) won't appear in My Tickets — this is intentional and documented

**Limitation:** Jira agents/admins could theoretically write to the field. This is acceptable — the threat model is external user spoofing, not insider attacks.

**Requires:** JSM admin to create the custom field and ensure it's excluded from all portal forms.

## Proxy Endpoints

### `GET /api/v1/my-requests`

List tickets belonging to the authenticated user.

#### Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-Acting-User` | Yes | ACCESS ID (e.g., `apasquale@access-ci.org`) — from MCP server's auth context |
| `X-Proxy-Secret` | Yes | Shared secret proving the request comes from an authorized MCP server |

The `X-Proxy-Secret` header prevents direct access to the proxy.

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | string | `open` | Filter by status: `open`, `closed`, `all` |
| `limit` | number | `20` | Maximum tickets to return (max 50) |
| `start` | number | `0` | Pagination offset |
| `ticket_key` | string | — | Look up a specific ticket by key (e.g., `ACCESS-12345`) |

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

### `GET /api/v1/requests/:key`

Get full details for a specific ticket including comments.

Same headers required. Proxy verifies the Verified Owner field matches `X-Acting-User` before returning data.

```json
{
  "success": true,
  "data": {
    "issueKey": "ACCESS-12345",
    "summary": "Cannot allocate GPUs on Delta",
    "description": "Full description text...",
    "status": "Waiting for support",
    "statusCategory": "open",
    "createdDate": "2024-01-15T10:30:00.000Z",
    "updatedDate": "2024-01-16T14:22:00.000Z",
    "requestType": "General Support",
    "priority": "Medium",
    "comments": [
      {
        "author": { "displayName": "Support Agent", "isStaff": true },
        "body": "Thanks for reporting this. We're looking into it.",
        "created": "2024-01-15T14:00:00.000Z"
      }
    ]
  }
}
```

## Security

### Proxy Authentication

Every ticket-reading request requires:

1. **`X-Proxy-Secret` matches `PROXY_SECRET` env var** — proves the request comes from an authorized MCP server. Return 403 if missing or wrong.

2. **`X-Acting-User` is a valid ACCESS ID format** — sanitize to `[a-zA-Z0-9@._-]` to prevent JQL injection.

### Jira Query (using Verified Owner field)

```
project = ACCESS AND cf[XXXXX] = "apasquale@access-ci.org" ORDER BY created DESC
```

Where `cf[XXXXX]` is the Verified Owner custom field ID.

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| User sees another user's tickets | Verified Owner field only set by authenticated channels; not on portal forms |
| Direct proxy access with forged headers | X-Proxy-Secret required |
| JQL injection | Input sanitization: `[a-zA-Z0-9@._-]` only |
| Unauthenticated MCP user | Tool checks `getRequestContext().actingUser` |
| Spoofed ACCESS ID on portal form | Verified Owner is a separate, hidden field |
| Legacy tickets without Verified Owner | Not returned — documented limitation |
| Atlassian OAuth for customers | Not feasible — unlicensed JSM users cannot complete OAuth consent |

## MCP Server Tools

### `get_my_tickets`

```typescript
{
  name: "get_my_tickets",
  description: "Get your support tickets from ACCESS. Identity determined automatically from authentication.",
  inputSchema: {
    type: "object",
    properties: {
      ticket_key: { type: "string", description: "Look up a specific ticket (e.g., ACCESS-12345)" },
      status: { type: "string", enum: ["open", "closed", "all"], default: "open" },
      limit: { type: "number", default: 10, description: "Max tickets to return (max 50)" }
    }
    // No required fields — no user identity parameters
  }
}
```

### `get_ticket_details`

```typescript
{
  name: "get_ticket_details",
  description: "Get full details of a support ticket including comments. Only returns tickets belonging to you.",
  inputSchema: {
    type: "object",
    properties: {
      ticket_key: { type: "string", description: "Ticket key (e.g., ACCESS-12345)" }
    },
    required: ["ticket_key"]
  }
}
```

## Implementation Prerequisites

Before building:

1. **JSM admin creates "Verified Owner" custom field** — text field, not on any portal form
2. **Update ticket creation flow** to populate Verified Owner with the authenticated ACCESS ID
3. **Stakeholder approval** on the Verified Owner approach and its limitations

## Implementation Steps (once approved)

### Phase 1: JSM Configuration
- Create Verified Owner custom field
- Note the custom field ID for JQL queries
- Verify it's not on any portal forms

### Phase 2: Update Ticket Creation
- Modify Netlify proxy `create-support-ticket` to set Verified Owner from `X-Acting-User` header
- Modify MCP JSM server to forward `actingUser` as header when creating tickets

### Phase 3: Netlify Proxy — Read Endpoints
- Create `my-requests` function
- Create `request-details` function
- Add PROXY_SECRET validation
- Add to `netlify.toml` redirects

### Phase 4: MCP Server — New Tools
- Add `get_my_tickets` and `get_ticket_details` to JSM server
- Add OAUTH_VERIFY_URL and PROXY_SECRET to docker-compose

### Phase 5: Testing
- Create ticket through authenticated channel → verify Verified Owner is set
- Query My Tickets → verify only verified tickets returned
- Verify portal-created tickets (no Verified Owner) don't appear
- Verify user A can't see user B's tickets
- Verify unauthenticated users get "Authentication required"

## Alternatives Considered

| Approach | Why Not |
|----------|---------|
| Atlassian OAuth 2.0 3LO | JSM customers (unlicensed) cannot complete OAuth consent on Jira Cloud |
| Query by reporter email | Reporter can be spoofed via email-submitted tickets or portal forms |
| Query by ACCESS ID custom field | User-editable form field — anyone can type another person's ID |
| Atlassian API tokens per user | Requires Atlassian account setup; customers can't generate tokens |
| Dual verification (ACCESS ID + email) | Both fields are user-supplied and spoofable |
