# MCP Authentication Architecture

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [MCP Action Tools](./05-events-actions.md) | [Announcements API Spec](./drupal-announcements-api-spec.md)

## Overview

This document describes how authenticated users can perform actions (create announcements, events, etc.) through the AI QA Bot system. The architecture ensures:

- User identity is verified through existing CILogon/ACCESS authentication
- Actions are attributed to the correct user for audit purposes
- User credentials are never exposed to intermediate systems
- MCP servers can make authorized API calls on behalf of users

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      AUTHENTICATION FLOW OVERVIEW                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  User    │    │  Drupal  │    │  QA Bot  │    │   MCP    │    │  Drupal  │ │
│   │ Browser  │    │  (Auth)  │    │          │    │  Server  │    │ JSON:API │ │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘ │
│        │               │               │               │               │        │
│   1. Login via CILogon │               │               │               │        │
│        │──────────────▶│               │               │               │        │
│        │               │               │               │               │        │
│   2. Drupal session    │               │               │               │        │
│        │◀──────────────│               │               │               │        │
│        │               │               │               │               │        │
│   3. Page with QA Bot + JWT            │               │               │        │
│        │◀──────────────│               │               │               │        │
│        │               │               │               │               │        │
│   4. User asks to create announcement  │               │               │        │
│        │───────────────────────────────▶               │               │        │
│        │               │               │               │               │        │
│   5. QA Bot sends request + JWT        │               │               │        │
│        │               │               │──────────────▶│               │        │
│        │               │               │               │               │        │
│   6. MCP validates JWT, extracts user  │               │               │        │
│        │               │               │               │               │        │
│   7. MCP calls JSON:API as service account + X-Acting-User header      │        │
│        │               │               │               │──────────────▶│        │
│        │               │               │               │               │        │
│   8. Drupal hook changes author to acting user, creates content        │        │
│        │               │               │               │◀──────────────│        │
│        │               │               │               │               │        │
│   9. Success response  │               │               │               │        │
│        │◀──────────────────────────────────────────────│               │        │
│        │               │               │               │               │        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Approach: Service Account + JSON:API

Instead of custom API endpoints, we use:
1. **Drupal JSON:API** for CRUD operations (standard Drupal)
2. **Key Auth module** for service account authentication
3. **Small custom module** (`access_mcp_author`) to swap author to acting user

This minimizes custom code while leveraging Drupal's built-in capabilities.

→ *See [Announcements API Spec](./drupal-announcements-api-spec.md) for implementation details*

---

## Key Principles

### 1. Service Account Pattern

The MCP server authenticates to Drupal as a service account (`mcp_service`), not as the end user:

| Component | Value |
|-----------|-------|
| Service account | `mcp_service` (Drupal user) |
| Authentication | Key Auth module (`api-key` header) |
| Acting user | Passed via `X-Acting-User` header |
| Author swap | `hook_node_presave()` in `access_mcp_author` module |

### 2. No Token Passthrough to Drupal

Per MCP specification, user tokens should NOT be forwarded to upstream APIs. Instead:

- MCP server validates the user's JWT
- MCP server authenticates to Drupal with its **own service account**
- User identity is passed as a **header** (not a credential)

### 3. Separation of Concerns

| System | Responsibility |
|--------|----------------|
| **Drupal (Auth)** | User login, session, JWT generation |
| **QA Bot** | Passes JWT with requests |
| **MCP Server** | Validates JWT, authenticates as service account, passes acting user |
| **Drupal (JSON:API)** | CRUD operations |
| **access_mcp_author** | Validates acting user, swaps author, enforces permissions |

---

## Components

### Component 1: Drupal JWT Generation

**Location**: New module or extension to existing auth

**Trigger**: When page with QA Bot renders for authenticated user

**JWT Claims**:

| Claim | Description | Example |
|-------|-------------|---------|
| `sub` | Drupal user ID | `12345` |
| `access_id` | User's ACCESS ID | `jsmith@access-ci.org` |
| `email` | User email | `jsmith@university.edu` |
| `name` | Display name | `Jane Smith` |
| `roles` | Drupal roles | `["authenticated", "announcement_creator"]` |
| `exp` | Expiration (1 hour) | `1733360400` |
| `iat` | Issued at | `1733356800` |
| `aud` | Audience | `mcp://actions` |
| `iss` | Issuer | `https://support.access-ci.org` |

**Signing**: RS256 (asymmetric) - Drupal holds private key, MCP servers have public key

### Component 2: QA Bot Token Handling

**Current state**: QA Bot receives user context props from Drupal:
- `isLoggedIn`
- `userEmail`
- `userName`
- `accessId`

**New**: Add `userToken` prop containing the JWT

**Behavior**:
- Include `userToken` in requests to n8n/MCP when performing authenticated actions
- Token is opaque to QA Bot - just passed through

### Component 3: MCP Server JWT Validation

**Location**: MCP server (e.g., `@access-mcp/announcements`)

**Validation steps**:
1. Verify JWT signature using Drupal's public key
2. Check `exp` claim (not expired)
3. Check `aud` claim (correct audience)
4. Check `iss` claim (correct issuer)
5. Extract `access_id` claim for the acting user

**On success**: Call Drupal JSON:API with service account credentials + `X-Acting-User` header

**On failure**: Return authentication error to QA Bot

### Component 4: Drupal Service Account + Key Auth

**Module**: [Key Auth](https://www.drupal.org/project/key_auth) (contrib)

**Setup**:
1. Create `mcp_service` user account with appropriate role
2. Configure Key Auth module for header-based authentication
3. Generate API key for the service account

**Request headers**:

| Header | Value | Purpose |
|--------|-------|---------|
| `api-key` | Service account key | Authenticates as `mcp_service` user |
| `X-Acting-User` | ACCESS ID | Identifies user to attribute action to |

### Component 5: Author Swap Module (`access_mcp_author`)

**Location**: `web/modules/custom/access_mcp_author/`

**Purpose**: Small module that intercepts requests from the service account and:
1. Reads `X-Acting-User` header
2. Looks up Drupal user by `field_access_id`
3. Validates acting user has permission for the action
4. Changes node owner to acting user

**Hooks**:
- `hook_node_presave()` - For create/update operations
- `hook_node_predelete()` - For delete operations

→ *See [Announcements API Spec](./drupal-announcements-api-spec.md) for implementation details*

---

## Drupal Implementation Requirements

### Key Auth Module (Contrib)

Install and configure:
```bash
composer require drupal/key_auth
drush en key_auth
```

Configure at `/admin/config/system/key-auth`:
- Enable header detection
- Set header name (e.g., `api-key`)

### Service Account Setup

Create a Drupal user:

| Field | Value |
|-------|-------|
| Username | `mcp_service` |
| Email | `mcp-service@access-ci.org` |
| Role | `mcp_service_role` (custom) |

Role permissions:
- `create access_news content`
- `edit any access_news content`
- `delete any access_news content`
- `use jsonapi`

### JWT Key Management

**Option A: Drupal Key Module**
- Store private key in Drupal's Key module
- Export public key for MCP servers
- Key rotation via Drupal admin

**Option B: External Key Management**
- Keys stored in environment/secrets manager
- Both Drupal and MCP servers reference same source
- Better for multi-environment deployments

**Recommendation**: Start with Drupal Key module for simplicity; migrate to external management if needed.

---

## Security Considerations

### Token Security

| Risk | Mitigation |
|------|------------|
| Token theft | Short expiry (1 hour), HTTPS only |
| Token replay | Include `iat` claim, consider nonce for sensitive ops |
| Privilege escalation | Roles embedded in token, validated server-side |
| Key compromise | Use asymmetric keys (RS256), rotate periodically |

### API Security

| Risk | Mitigation |
|------|------------|
| API key exposure | Store in Drupal Key module, never in client code; single key simplifies rotation |
| Unauthorized access | Validate both API key AND user identity |
| CSRF | API endpoints are stateless, no session cookies |
| Rate limiting | Implement per-user rate limits |

### Audit Trail

All authenticated API operations must log:
- Timestamp
- User ACCESS ID (from `X-Acting-User`)
- Action performed
- Entity affected (ID, type)
- Success/failure
- Calling service (from API key)

---

## Implementation Phases

### Phase 1: Drupal Setup

**Scope**: Configure Drupal for JSON:API writes with service account

- [ ] Install Key Auth module
- [ ] Create `mcp_service` user account
- [ ] Create `mcp_service_role` with permissions
- [ ] Generate API key for service account
- [ ] Enable JSON:API writes (`read_only: false`)
- [ ] Test JSON:API create/update/delete with service account

**Deliverable**: Service account can create content via JSON:API

### Phase 2: Author Swap Module

**Scope**: Create `access_mcp_author` module

- [ ] Implement `hook_node_presave()` for author swap
- [ ] Implement `hook_node_predelete()` for delete validation
- [ ] Add validation (tags, coordinator permission, ownership)
- [ ] Add logging for audit trail
- [ ] Unit tests

**Deliverable**: Requests from service account correctly attributed to acting user

### Phase 3: JWT Integration

**Scope**: Generate and pass JWT to QA Bot

- [ ] JWT generation service in Drupal
- [ ] Add JWT to QA Bot page context
- [ ] Update QA Bot to pass token with requests
- [ ] Test token flow end-to-end

**Deliverable**: JWT available in QA Bot for authenticated users

### Phase 4: MCP Server Integration

**Scope**: MCP server validates JWT and calls Drupal JSON:API

- [ ] Add JWT validation to MCP server
- [ ] Add JSON:API client for Drupal calls
- [ ] Pass `X-Acting-User` header from validated JWT
- [ ] End-to-end test: QA Bot → MCP → Drupal

**Deliverable**: Complete authentication flow working

---

## Configuration Checklist

### Drupal Configuration

- [ ] Key Auth module installed and configured
- [ ] `mcp_service` user account created
- [ ] `mcp_service_role` role with permissions
- [ ] API key generated for service account
- [ ] JSON:API writes enabled
- [ ] `access_mcp_author` module enabled
- [ ] JWT private key stored in Key module
- [ ] QA Bot page includes JWT in user context

### MCP Server Configuration

Environment variables needed (shared across all MCP servers):

| Variable | Description |
|----------|-------------|
| `DRUPAL_JWT_PUBLIC_KEY` | PEM-encoded public key for JWT validation |
| `DRUPAL_SERVICE_ACCOUNT_KEY` | API key for `mcp_service` user |
| `DRUPAL_BASE_URL` | Base URL for Drupal JSON:API calls |
| `JWT_AUDIENCE` | Expected audience claim (e.g., `mcp://actions`) |
| `JWT_ISSUER` | Expected issuer claim (e.g., `https://support.access-ci.org`) |

---

## Open Questions

### Technical

1. **Key rotation**: What's the process for rotating JWT signing keys?
2. **Token refresh**: Should tokens be refreshable, or require page reload?
3. **Audience granularity**: One audience for all MCP actions, or per-domain (e.g., `mcp://announcements`, `mcp://events`)?

### Operational

4. **Key distribution**: How do MCP servers get the public key? Manual config, or fetch from Drupal endpoint?
5. **Monitoring**: What alerts for auth failures?

---

## Glossary

| Term | Definition |
|------|------------|
| **JWT** | JSON Web Token - signed token containing user claims |
| **ACCESS ID** | User's identifier in ACCESS system (e.g., `user@access-ci.org`) |
| **Service Key** | API key that identifies a trusted service (MCP server) |
| **Acting User** | The user on whose behalf an action is performed |
| **MCP** | Model Context Protocol - standard for AI tool interactions |

---

## References

- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- [RFC 7519 - JSON Web Token](https://tools.ietf.org/html/rfc7519)
- [Drupal Key Module](https://www.drupal.org/project/key)
- [Firebase JWT PHP Library](https://github.com/firebase/php-jwt)
