# Backend Integration Specification

> **Related**: [MCP Authentication](./06-mcp-authentication.md) | [Agent Architecture](./01-agent-architecture.md)

## Overview

This document defines the standard pattern for ACCESS MCP servers to authenticate with backend APIs on behalf of users. It serves as:

1. **Implementation guide** for the MCP server team
2. **Integration contract** for backend API teams

The goal: any ACCESS backend API can integrate with the MCP server using a consistent, secure pattern.

---

## Design Principles

1. **MCP server is the trust boundary** - Backends trust the MCP server, not end-user tokens
2. **Service-to-service authentication** - MCP authenticates to backends with its own credentials
3. **User identity passed, not impersonated** - Acting user declared via header, not credential
4. **Backend owns authorization** - Each backend decides what the acting user can do
5. **ACCESS ID is canonical** - All systems use `access_id` (= CILogon eppn) as user identifier

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AUTHENTICATION FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐         ┌──────────────────┐         ┌──────────────────┐        │
│   │   MCP    │         │   ACCESS MCP     │         │   Backend API    │        │
│   │  Client  │         │     Server       │         │  (any service)   │        │
│   └────┬─────┘         └────────┬─────────┘         └────────┬─────────┘        │
│        │                        │                            │                  │
│   1. Tool call + Bearer token   │                            │                  │
│        │───────────────────────▶│                            │                  │
│        │                        │                            │                  │
│   2. Validate MCP token         │                            │                  │
│        │                        │ (extract access_id)        │                  │
│        │                        │                            │                  │
│   3. Call backend API           │                            │                  │
│        │                        │───────────────────────────▶│                  │
│        │                        │  Authorization: Bearer {service_token}        │
│        │                        │  X-Acting-User: {access_id}                   │
│        │                        │                            │                  │
│   4. Backend validates          │                            │                  │
│        │                        │                 • Verify service_token        │
│        │                        │                 • Parse X-Acting-User         │
│        │                        │                 • Apply authorization         │
│        │                        │                            │                  │
│   5. Response                   │                            │                  │
│        │                        │◀───────────────────────────│                  │
│        │◀───────────────────────│                            │                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## MCP Server Responsibilities

### 1. User Authentication

The MCP server authenticates users via OAuth 2.1 with CILogon:

- Acts as OAuth authorization server to MCP clients
- Proxies authentication to CILogon
- Issues MCP access tokens containing user identity

### 2. Token Validation

For each tool call, the MCP server:

1. Validates the Bearer token (signature, expiration, audience)
2. Extracts `access_id` claim
3. Determines required scopes for the operation

### 3. Backend Calls

When calling a backend API:

1. Authenticate using the backend's service credential
2. Include `X-Acting-User` header with the user's `access_id`
3. Handle errors and map to appropriate MCP responses

---

## Backend API Contract

### Required Headers

Every request from the MCP server includes:

| Header | Value | Required | Purpose |
|--------|-------|----------|---------|
| `Authorization` | `Bearer {service_token}` | Always | Authenticates MCP server |
| `X-Acting-User` | `{access_id}` | For user-scoped operations | Identifies the user |
| `X-Request-ID` | `{uuid}` | Always | Correlation ID for tracing |

### Service Token

The `service_token` is a credential that identifies the ACCESS MCP server. Options:

| Type | Format | Use Case |
|------|--------|----------|
| **API Key** | Static string | Simple backends, internal services |
| **JWT** | Signed token with claims | Services needing MCP server identity claims |
| **OAuth Client Credentials** | Token from OAuth flow | Services with existing OAuth infrastructure |

**Recommendation**: Start with API keys for simplicity. Migrate to OAuth client credentials if backends already support it.

### X-Acting-User Header

```
X-Acting-User: jsmith@access-ci.org
```

- Value is the user's ACCESS ID (= CILogon eppn)
- Present for all user-scoped operations
- Absent for purely public/anonymous operations
- Backend MUST validate this is a valid ACCESS ID format

**Trust boundary**: This header is ONLY set by trusted services (the agent or MCP server) after validating the user's identity server-side. It is never passed through from client-supplied values.

| Source | How Identity Is Validated | Who Sets X-Acting-User |
|--------|--------------------------|----------------------|
| Direct MCP client (Claude, ChatGPT) | OAuth 2.1 token validated by MCP server | MCP server |
| Browser QA Bot | JWT cookie validated by agent | Agent |

Backends MUST only accept `X-Acting-User` from authenticated service accounts (validated via `Authorization` header). Any request without a valid service token should be rejected regardless of `X-Acting-User` presence.

### X-Request-ID Header

```
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

- UUID v4 generated by MCP server
- Include in all log entries for request correlation
- Return in error responses for debugging

---

## Backend Implementation Guide

### Step 1: Register Service Credential

Create a credential for the ACCESS MCP server:

```
# Example: API key approach
Service Name: access-mcp-server
API Key: [generated secure key]
Permissions: [define based on operations needed]
```

Store the credential securely. Provide to MCP server team via secure channel.

### Step 2: Implement Authentication Middleware

Validate the service token on every request:

```python
# Pseudocode - adapt to your framework

def authenticate_mcp_request(request):
    # 1. Extract and validate service token
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return error(401, 'Missing or invalid Authorization header')

    service_token = auth_header[7:]  # Remove 'Bearer ' prefix

    if not validate_service_token(service_token):
        return error(401, 'Invalid service token')

    # 2. Extract acting user (if present)
    acting_user = request.headers.get('X-Acting-User')

    if acting_user:
        if not is_valid_access_id_format(acting_user):
            return error(400, 'Invalid X-Acting-User format')

        request.acting_user = acting_user
    else:
        request.acting_user = None

    # 3. Store request ID for logging
    request.request_id = request.headers.get('X-Request-ID', generate_uuid())

    return proceed()
```

### Step 3: Implement Authorization

Authorization depends on the operation type:

#### Public Read Operations

No `X-Acting-User` required. Service token is sufficient.

```python
@route('/api/resources')
def list_resources(request):
    # Service token validated by middleware
    # No user context needed - return public data
    return get_all_resources()
```

#### User-Scoped Read Operations

`X-Acting-User` required. Filter results to that user's data.

```python
@route('/api/allocations')
def list_allocations(request):
    if not request.acting_user:
        return error(400, 'X-Acting-User required for this endpoint')

    # Return only this user's allocations
    return get_allocations_for_user(request.acting_user)
```

#### Write Operations

`X-Acting-User` required. Check permissions before allowing action.

```python
@route('/api/announcements', methods=['POST'])
def create_announcement(request):
    if not request.acting_user:
        return error(400, 'X-Acting-User required for this endpoint')

    # Look up user and check permissions
    user = lookup_user_by_access_id(request.acting_user)
    if not user:
        return error(403, 'User not found')

    if not user.has_permission('create_announcement'):
        return error(403, 'User lacks permission to create announcements')

    # Create with proper attribution
    announcement = create_announcement(
        data=request.body,
        author=user,
        created_by_service='access-mcp-server'
    )

    return success(announcement)
```

### Step 4: Implement Audit Logging

Log all operations with consistent fields:

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "service": "access-mcp-server",
  "acting_user": "jsmith@access-ci.org",
  "action": "create_announcement",
  "resource_type": "announcement",
  "resource_id": "12345",
  "result": "success",
  "ip_address": "192.168.1.1"
}
```

Required fields:

| Field | Description |
|-------|-------------|
| `timestamp` | ISO 8601 timestamp |
| `request_id` | From `X-Request-ID` header |
| `service` | Always "access-mcp-server" for MCP requests |
| `acting_user` | From `X-Acting-User` header (null if anonymous) |
| `action` | Operation performed |
| `resource_type` | Type of resource affected |
| `resource_id` | ID of resource affected (if applicable) |
| `result` | "success" or "failure" |

### Step 5: Error Responses

Return errors in a consistent format:

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "User lacks permission to create announcements",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

Standard error codes:

| HTTP Status | Code | When to Use |
|-------------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid request format, missing required header |
| 401 | `UNAUTHORIZED` | Invalid or missing service token |
| 403 | `FORBIDDEN` | Valid auth but user lacks permission |
| 404 | `NOT_FOUND` | Resource doesn't exist |
| 422 | `VALIDATION_ERROR` | Request body fails validation |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |

---

## Authorization Patterns

### Pattern 1: Public Data

No user context needed. Anyone can read.

```
Examples: Compute resources, software catalog, NSF awards
X-Acting-User: Not required
Authorization: Service token only
```

### Pattern 2: User-Owned Data

User can only access their own data.

```
Examples: User's allocations, user's usage metrics
X-Acting-User: Required
Authorization: Filter results to acting_user only
```

### Pattern 3: Role-Based Access

User needs specific role/permission to perform action.

```
Examples: Create announcement, manage affinity group
X-Acting-User: Required
Authorization: Check user has required role in your system
```

### Pattern 4: Ownership + Admin

User can modify their own resources, admins can modify any.

```
Examples: Update own announcement, delete own event
X-Acting-User: Required
Authorization:
  - If user owns resource: allow
  - If user is admin: allow
  - Otherwise: deny
```

---

## ACCESS ID Format

The `access_id` follows CILogon eppn format:

```
{username}@{scope}
```

Examples:
- `jsmith@access-ci.org`
- `researcher@university.edu`

Validation regex:
```regex
^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
```

---

## Security Considerations

### Service Token Security

| Risk | Mitigation |
|------|------------|
| Token theft | Rotate tokens periodically; use short-lived JWTs if possible |
| Token in logs | Never log the full token; log only a prefix for debugging |
| Unauthorized services | Maintain allowlist of valid service tokens |

### Acting User Security

| Risk | Mitigation |
|------|------------|
| Spoofed header from client | `X-Acting-User` is NEVER passed through from clients; always extracted server-side from validated JWT cookie (QA Bot) or OAuth token (direct MCP client) |
| Spoofed header from unauthorized service | Only accept `X-Acting-User` from requests with a valid service token in the `Authorization` header |
| Invalid user | Validate `access_id` format; optionally verify user exists |
| Privilege escalation | Always check permissions in your system, not just presence of header |

### Rate Limiting

Implement rate limits to prevent abuse:

| Level | Recommendation |
|-------|----------------|
| Per service | 10,000 requests/hour (high - it's server-to-server) |
| Per acting user | 100 requests/hour (prevents runaway AI loops) |

Return `429 Too Many Requests` with `Retry-After` header when exceeded.

---

## Backend Onboarding Checklist

For each backend API integrating with the MCP server:

### Prerequisites

- [ ] API supports HTTPS (required)
- [ ] API can add custom header parsing
- [ ] Team available for integration support

### Implementation

- [ ] Generate service credential for MCP server
- [ ] Implement `Authorization` header validation
- [ ] Implement `X-Acting-User` header parsing
- [ ] Implement `X-Request-ID` header handling
- [ ] Implement authorization logic per endpoint
- [ ] Implement audit logging
- [ ] Implement rate limiting
- [ ] Document endpoints and authorization rules

### Testing

- [ ] Test with valid service token
- [ ] Test with invalid/missing service token (expect 401)
- [ ] Test with valid `X-Acting-User`
- [ ] Test with invalid `X-Acting-User` format (expect 400)
- [ ] Test authorization rules (expect 403 when appropriate)
- [ ] Verify audit logs capture all fields

### Handoff

- [ ] Provide service credential to MCP server team (secure channel)
- [ ] Document base URL and available endpoints
- [ ] Document authorization rules for each endpoint
- [ ] Provide test/staging environment access

---

## MCP Server Configuration

The MCP server maintains configuration for each backend:

```yaml
backends:
  drupal:
    name: "Drupal CMS"
    base_url: https://support.access-ci.org
    service_token_env: DRUPAL_SERVICE_TOKEN
    endpoints:
      - path: /jsonapi/node/access_news
        methods: [GET, POST, PATCH, DELETE]
        auth_pattern: role_based

  xdmod:
    name: "XDMoD Metrics"
    base_url: https://xdmod.access-ci.org
    service_token_env: XDMOD_SERVICE_TOKEN
    endpoints:
      - path: /api/metrics
        methods: [GET]
        auth_pattern: user_scoped

  allocations:
    name: "Allocations Service"
    base_url: https://allocations.access-ci.org
    service_token_env: ALLOCATIONS_SERVICE_TOKEN
    endpoints:
      - path: /api/allocations
        methods: [GET]
        auth_pattern: user_scoped
      - path: /api/projects
        methods: [GET]
        auth_pattern: public

  # Additional backends...
```

---

## Timeline and Rollout

### Phase 1: Specification

- [x] Define backend integration contract (this document)
- [ ] Review with backend API teams
- [ ] Finalize header names and error formats

### Phase 2: MCP Server Implementation

- [ ] Implement service token management
- [ ] Implement `X-Acting-User` header injection
- [ ] Implement `X-Request-ID` generation and logging
- [ ] Implement backend configuration system

### Phase 3: Drupal Integration (Pilot)

- [ ] Drupal implements contract for JSON:API
- [ ] End-to-end testing with announcements
- [ ] Document lessons learned

### Phase 4: Remaining Backends

- [ ] XDMoD integration
- [ ] Allocations integration
- [ ] System status integration
- [ ] Software discovery integration
- [ ] Remaining backends

---

## Open Questions

| Question | Options | Recommendation |
|----------|---------|----------------|
| Service token format | API key vs JWT vs OAuth | Start with API keys; migrate if needed |
| Token rotation | Manual vs automated | Automated with 90-day rotation |
| User validation | Format only vs existence check | Format only (backends know their users) |
| Rate limit scope | Per-service vs per-user vs both | Both |

---

## References

- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- [RFC 7235 - HTTP Authentication](https://tools.ietf.org/html/rfc7235)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
