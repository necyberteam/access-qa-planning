# MCP Authentication Architecture

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [Events Actions](./05-events-actions.md) | [Announcements API Spec](./drupal-announcements-api-spec.md)

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
│   │ Browser  │    │  (Auth)  │    │          │    │  Server  │    │  (API)   │ │
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
│   7. MCP calls Drupal API with service key + user ID   │               │        │
│        │               │               │               │──────────────▶│        │
│        │               │               │               │               │        │
│   8. Drupal creates content as user    │               │               │        │
│        │               │               │               │◀──────────────│        │
│        │               │               │               │               │        │
│   9. Success response  │               │               │               │        │
│        │◀──────────────────────────────────────────────│               │        │
│        │               │               │               │               │        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Principles

### 1. No Token Passthrough

Per MCP specification, user tokens should NOT be forwarded to upstream APIs. Instead:

- MCP server validates the user's JWT
- MCP server uses its **own service credentials** to call Drupal
- User identity is passed as a **claim** (not a credential)

### 2. Trusted Intermediary Pattern

The MCP server acts as a trusted intermediary:

| What MCP Server Does | Why |
|---------------------|-----|
| Validates user JWT | Confirms user identity |
| Uses own API key | Drupal trusts the MCP server |
| Passes user ID as header | Drupal knows who to attribute action to |

### 3. Separation of Concerns

| System | Responsibility |
|--------|----------------|
| **Drupal (Auth)** | User login, session, JWT generation |
| **QA Bot** | Passes JWT with requests |
| **MCP Server** | Validates JWT, makes authorized API calls |
| **Drupal (API)** | Validates service key, performs action as user |

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
5. Extract user claims for authorization

**On success**: Proceed with API call using extracted user identity

**On failure**: Return authentication error to QA Bot

### Component 4: Drupal API Authentication

**Location**: API modules (e.g., `access_announcements_api`)

**Headers required**:

| Header | Value | Purpose |
|--------|-------|---------|
| `X-API-Key` | Service key | Identifies trusted caller (MCP server) |
| `X-Acting-User` | ACCESS ID | Identifies user performing action |

**Validation steps**:
1. Validate `X-API-Key` against stored service key
2. Look up Drupal user by `field_access_id` matching `X-Acting-User`
3. Verify user exists and is active
4. Perform action with that user as owner

---

## Drupal Implementation Requirements

### New Module: `access_mcp_auth`

A shared authentication module that can be used by all MCP-facing API modules.

**Provides**:
1. JWT generation service for authenticated pages
2. API key validation service
3. User lookup by ACCESS ID
4. Shared middleware for MCP API endpoints

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

### Service API Key Management

A **single shared service key** is used by all MCP servers:

| Key ID | Purpose | Used By |
|--------|---------|---------|
| `mcp_service_key` | All MCP API calls | All MCP servers |

**Why a single key (not per-server)?**
- All MCP servers run in the same trusted environment (n8n)
- User identity (`X-Acting-User`) provides per-action authorization
- The API key just validates "this is trusted MCP infrastructure"
- Simpler to manage, deploy, and rotate

Store via Drupal Key module. Rotate periodically.

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

### Phase 1: Drupal Foundation

**Scope**: Create `access_mcp_auth` module with core services

- [ ] JWT generation service
- [ ] API key validation service
- [ ] User lookup by ACCESS ID
- [ ] Key configuration via Key module
- [ ] Unit tests

**Deliverable**: Shared auth infrastructure for MCP APIs

### Phase 2: JWT Integration with QA Bot

**Scope**: Generate and pass JWT to QA Bot

- [ ] Add JWT to QA Bot page context (drupalSettings or prop)
- [ ] Update QA Bot to accept and pass token
- [ ] Test token flow end-to-end (without MCP integration)

**Deliverable**: JWT available in QA Bot for authenticated users

### Phase 3: First API Module (Announcements)

**Scope**: Use auth infrastructure in Announcements API

- [ ] `access_announcements_api` uses `access_mcp_auth` services
- [ ] Integration test with mock MCP calls
- [ ] Deploy to staging

**Deliverable**: Working authenticated API for announcements

### Phase 4: MCP Server Integration

**Scope**: MCP server validates JWT and calls Drupal API

- [ ] Add JWT validation to announcements MCP server
- [ ] Configure public key and API key
- [ ] End-to-end test: QA Bot → MCP → Drupal

**Deliverable**: Complete authentication flow working

---

## Configuration Checklist

### Drupal Configuration

- [ ] JWT private key stored in Key module
- [ ] Service API key(s) stored in Key module
- [ ] `access_mcp_auth` module enabled
- [ ] QA Bot page includes JWT in user context

### MCP Server Configuration

Environment variables needed (shared across all MCP servers):

| Variable | Description |
|----------|-------------|
| `DRUPAL_JWT_PUBLIC_KEY` | PEM-encoded public key for JWT validation |
| `MCP_SERVICE_KEY` | Shared service API key for calling Drupal APIs |
| `DRUPAL_BASE_URL` | Base URL for Drupal API calls |
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
