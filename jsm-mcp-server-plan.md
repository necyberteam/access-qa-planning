# JSM MCP Server Plan

## Overview

Add a new MCP server to `access_mcp` that enables the ACCESS agent to create support tickets in Jira Service Management (JSM) via the existing Netlify proxy.

## Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  access-agent   │────▶│  jsm MCP server │────▶│  Netlify Proxy  │────▶│  Atlassian JSM  │
│  (LangGraph)    │     │  (TypeScript)   │     │  (existing)     │     │  (Cloud)        │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                              │                  access-jsm-api.
                              │                  netlify.app
                              ▼
                        API key required
                        for write operations
```

## Why This Approach

1. **Reuse existing infrastructure** - Netlify proxy already handles JSM auth, field mapping, and ProForma integration
2. **No credential management in MCP** - Service account credentials stay in Netlify
3. **Consistent patterns** - Follows same structure as other MCP servers in the repo
4. **Battle-tested** - The proxy has been working in qa-bot

## Security: API Key Authentication

All MCP servers that support write operations (via `X-Acting-User`) will require API key authentication.

### Why

- MCP servers are called via HTTP
- Without auth, anyone who can reach the server could create tickets/content impersonating users
- Network isolation helps but defense in depth is better

### Implementation

Add to `BaseAccessServer` in shared package:

```typescript
// Constructor option
constructor(serverName, version, baseURL, { requireApiKey: true })

// Middleware checks X-Api-Key header
if (requireApiKey && req.header("X-Api-Key") !== process.env.MCP_API_KEY) {
  return res.status(401).json({ error: "Invalid or missing API key" });
}
```

### Servers Requiring API Key

| Server | Reason |
|--------|--------|
| jsm | Creates tickets on behalf of users |
| announcements | Creates/updates content on behalf of users |
| events | Creates/updates content on behalf of users |

### Configuration

```bash
# On MCP servers
MCP_API_KEY=<shared secret>

# On access-agent
MCP_API_KEY=<same shared secret>
```

## Netlify Proxy API Contract

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/tickets` | POST | Create support tickets (Service Desk 2) |
| `/api/v1/security-incidents` | POST | Create security incidents (Service Desk 3) |

### Request Body

```json
{
  "serviceDeskId": 2,
  "requestTypeId": 17,
  "requestFieldValues": {
    "email": "user@example.com",
    "name": "User Name",
    "summary": "Ticket title",
    "description": "Detailed description",
    "accessId": "username",
    "priority": "medium",
    "issueType": "User Account Question",
    "hasResourceProblem": "Yes",
    "resourceName": "Delta",
    "userIdAtResource": "jsmith",
    "keywords": "GPU, CUDA",
    "suggestedKeyword": ""
  }
}
```

### Response

```json
{
  "success": true,
  "data": {
    "ticketKey": "SUPPORT-12345",
    "ticketUrl": "https://access-ci.atlassian.net/servicedesk/customer/portal/2/SUPPORT-12345"
  }
}
```

## Service Desk Configuration

| Service Desk | ID | Purpose |
|--------------|-----|---------|
| ACCESS Support | 2 | General support, login help, feedback |
| ACCESS Cybersecurity | 3 | Security incidents |

## Request Types

| Type | ID | Service Desk | Use Case |
|------|-----|--------------|----------|
| General Support | 17 | 2 | General help, feedback |
| ACCESS Login | 30 | 2 | ACCESS website login issues |
| Resource Provider Login | 31 | 2 | RP-specific login issues |
| Security Incident | 26 | 3 | Security vulnerabilities/incidents |

## MCP Tools

### 1. `create_support_ticket`

Create a general support ticket.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "summary": {
      "type": "string",
      "description": "Brief title for the ticket"
    },
    "description": {
      "type": "string",
      "description": "Detailed description of the issue including what was tried"
    },
    "user_email": {
      "type": "string",
      "description": "User's email address"
    },
    "user_name": {
      "type": "string",
      "description": "User's full name"
    },
    "access_id": {
      "type": "string",
      "description": "User's ACCESS ID (optional)"
    },
    "category": {
      "type": "string",
      "enum": ["User Account Question", "Allocation Question", "User Support Question", "Training Question", "Metrics Question", "OnDemand Question", "XDMoD Question", "Some Other Question"],
      "description": "Issue category"
    },
    "priority": {
      "type": "string",
      "enum": ["lowest", "low", "medium", "high", "highest"],
      "default": "medium"
    },
    "resource": {
      "type": "string",
      "description": "ACCESS resource involved (if any)"
    },
    "user_id_at_resource": {
      "type": "string",
      "description": "User's ID at the resource (if applicable)"
    }
  },
  "required": ["summary", "description", "user_email", "user_name"]
}
```

### 2. `create_login_ticket`

Create a ticket for login issues.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "summary": {
      "type": "string",
      "description": "Brief description of login issue"
    },
    "description": {
      "type": "string",
      "description": "Detailed description"
    },
    "user_email": { "type": "string" },
    "user_name": { "type": "string" },
    "login_type": {
      "type": "string",
      "enum": ["access_portal", "resource_provider"],
      "description": "Where the login issue is occurring"
    },
    "identity_provider": {
      "type": "string",
      "enum": ["ACCESS", "Github", "Google", "Institution", "Microsoft", "ORCID", "Other"],
      "description": "For ACCESS portal login issues"
    },
    "resource": {
      "type": "string",
      "description": "For resource provider login issues"
    },
    "user_id_at_resource": {
      "type": "string",
      "description": "Username at the resource"
    }
  },
  "required": ["summary", "description", "user_email", "user_name", "login_type"]
}
```

### 3. `report_security_incident`

Report a security vulnerability or incident.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "summary": {
      "type": "string",
      "description": "Brief description of security concern"
    },
    "description": {
      "type": "string",
      "description": "Detailed description"
    },
    "user_email": { "type": "string" },
    "user_name": { "type": "string" },
    "access_id": { "type": "string" },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high", "critical"],
      "default": "medium"
    }
  },
  "required": ["summary", "description", "user_email", "user_name"]
}
```

### 4. `get_ticket_types`

Returns available ticket types and categories (read-only helper).

**Input:** None

**Output:**
```json
{
  "ticket_types": [
    {
      "type": "support",
      "name": "General Support",
      "description": "General support questions about allocations, accounts, software, etc.",
      "categories": ["User Account Question", "Allocation Question", ...]
    },
    {
      "type": "login_access",
      "name": "ACCESS Portal Login",
      "description": "Issues logging into the ACCESS portal"
    },
    {
      "type": "login_provider",
      "name": "Resource Provider Login",
      "description": "Issues logging into a specific resource provider"
    },
    {
      "type": "security",
      "name": "Security Incident",
      "description": "Security vulnerabilities or incidents"
    }
  ]
}
```

## Agent Integration

### User Context

The agent already has user info from CILogon authentication:
- Email
- Name
- ACCESS ID

These are passed to the MCP tool - no need to ask the user.

### Conversation Flow: Contextual Handoff

The agent tries to help first, then offers ticket creation when appropriate:

```
User: "I can't SSH into Expanse"
Agent: [troubleshooting questions and suggestions]
User: "I tried that, still doesn't work"
Agent: "I can create a support ticket for you. Based on our conversation,
        here's what I'd include:

        Title: Cannot SSH to Expanse - connection timeout
        Description: User cannot connect via SSH to Expanse. Getting
        connection timeout error. Was working last week. Already tried
        verifying hostname and regenerating SSH key.
        Resource: Expanse

        Would you like me to create this ticket? You can also suggest
        changes to the summary."
User: "Yes, create it"
Agent: [calls create_support_ticket]
Agent: "Done! Your ticket is SUPPORT-12345: [link]"
```

### Key Design Decisions

1. **Summary only, no transcript** - Users should feel safe asking "dumb questions" without worry that support staff will see their exact words. The agent writes a clean technical summary.

2. **User confirmation required** - Agent shows preview of ticket before creating. User can modify or cancel.

3. **Agent writes description** - Based on conversation context, the agent crafts a description that includes the issue and what was already tried. No raw conversation attached.

4. **Structured fields extracted** - Resource, category, priority are extracted and mapped to JSM ProForma fields where applicable.

### When to Offer Ticket Creation

To be tuned based on usage, but initial triggers:
- Agent can't answer the question (no RAG match, no tool can help)
- User expresses frustration or says something isn't working after troubleshooting
- User explicitly asks for human help or to open a ticket
- Issue requires account-specific investigation (allocations, permissions)

## Package Structure

```
packages/jsm/
├── src/
│   ├── index.ts          # Entry point
│   └── server.ts         # JsmServer class
├── package.json
├── tsconfig.json
└── README.md
```

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `JSM_PROXY_URL` | Yes | Netlify proxy base URL |
| `MCP_API_KEY` | Yes | Shared secret for agent authentication |

Default proxy URL: `https://access-jsm-api.netlify.app`

### Docker Compose Addition

```yaml
mcp-jsm:
  build:
    context: .
    dockerfile: Dockerfile.node
    args:
      PACKAGE: jsm
  environment:
    - PORT=3012
    - JSM_PROXY_URL=https://access-jsm-api.netlify.app
    - MCP_API_KEY=${MCP_API_KEY}
  ports:
    - "3012:3000"
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
    interval: 30s
    timeout: 10s
    retries: 3
```

## Implementation Steps

### Phase 1: Shared Package Update
- [ ] Add `requireApiKey` option to `BaseAccessServer`
- [ ] Add API key validation middleware to `/tools/:toolName` endpoint
- [ ] Update announcements server to enable API key requirement

### Phase 2: JSM MCP Server
- [ ] Create `packages/jsm/` directory structure
- [ ] Implement `JsmServer` class extending `BaseAccessServer`
- [ ] Implement tools: `create_support_ticket`, `create_login_ticket`, `report_security_incident`, `get_ticket_types`
- [ ] Add to `docker-compose.yml`
- [ ] Write README

### Phase 3: Agent Integration
- [ ] Add JSM server to MCP catalog configuration
- [ ] Update agent to pass API key to write-operation servers
- [ ] Update system prompt to handle ticket creation flow
- [ ] Test end-to-end ticket creation

### Phase 4: Testing & Documentation
- [ ] Test all ticket types create correctly in JSM
- [ ] Test error handling (invalid fields, proxy down, etc.)
- [ ] Test API key rejection for unauthorized requests
- [ ] Document in access-agent README

## Success Criteria

- [ ] Agent can create support tickets via natural conversation
- [ ] Tickets appear in JSM with correct fields populated
- [ ] User sees ticket preview and confirms before creation
- [ ] User info (email, name) comes from auth context, not asked again
- [ ] API key required - unauthorized requests rejected
- [ ] Error handling provides useful feedback to users
- [ ] No credentials exposed in MCP server

## Open Questions

1. **Rate limiting** - Should we add rate limiting to prevent accidental ticket spam? (Probably not for v1, monitor usage)

2. **Attachments** - The proxy supports file attachments but MCP doesn't handle binary well. Skip for v1?

3. **Ticket lookup** - Should users be able to check status of their tickets? (Future enhancement)
