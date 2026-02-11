# MCP Authentication Architecture (OAuth 2.1)

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [MCP Action Tools](./05-events-actions.md) | [Backend Integration Spec](./07-backend-integration-spec.md) | [Announcements API Spec](./drupal-announcements-api-spec.md) | [QA Bot Authentication](./08-qa-bot-authentication.md)

## Overview

This document describes OAuth 2.1 authentication for **direct MCP client connections** — researchers connecting from Claude, ChatGPT, Copilot, and other MCP-compatible clients.

> **Important**: This OAuth flow applies only to direct MCP clients. The browser-based QA Bot uses a different authentication model — a signed JWT cookie scoped to `.access-ci.org`. See [QA Bot Authentication](./08-qa-bot-authentication.md) for that flow. The two authentication paths converge at the same point: the MCP server receives a validated user identity and passes `X-Acting-User` to backend APIs.

The architecture ensures:

- User identity verified through CILogon
- Actions attributed to the correct user for audit purposes
- Standards-compliant OAuth 2.1 flow per MCP specification
- Single redirect URI simplifies CILogon registration
- No local installation required for end users

### Two Authentication Paths, One Backend Pattern

```
Direct MCP Clients (Claude, ChatGPT, etc.)
  └─→ OAuth 2.1 with CILogon (this document)
  └─→ MCP server validates token, extracts ACCESS ID
  └─→ Calls backends with X-Acting-User header

Browser-Based QA Bot
  └─→ JWT cookie on .access-ci.org (see 08-qa-bot-authentication.md)
  └─→ Agent validates cookie, extracts ACCESS ID
  └─→ Calls MCP servers / backends with X-Acting-User header

Both paths result in the same backend call pattern:
  Authorization: Bearer {service_token}
  X-Acting-User: {access_id}
  X-Request-ID: {uuid}
```

---

## Use Case

Researchers connect their AI tools to ACCESS-CI infrastructure:

1. User adds `https://mcp.access-ci.org/mcp` to Claude, ChatGPT, or other MCP client
2. MCP client initiates OAuth flow with ACCESS MCP server
3. ACCESS MCP server redirects user to CILogon for authentication
4. User authenticates via their institution (4000+ supported)
5. User can now query ACCESS software, resources, allocations, XDMoD data
6. Authorized users can create/edit announcements and events

---

## Authentication Flow

The MCP server acts as an OAuth authorization server to MCP clients while delegating actual authentication to CILogon. This proxy pattern is the standard approach for MCP servers using third-party identity providers.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         OAUTH PROXY AUTHENTICATION FLOW                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐  │
│   │   MCP    │         │  ACCESS  │         │ CILogon  │         │ Backend  │  │
│   │  Client  │         │   MCP    │         │  (IdP)   │         │   APIs   │  │
│   │          │         │  Server  │         │          │         │          │  │
│   └────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘  │
│        │                    │                    │                    │         │
│   1. User adds MCP server URL                    │                    │         │
│        │                    │                    │                    │         │
│   2. Client fetches OAuth metadata               │                    │         │
│        │───────────────────▶│                    │                    │         │
│        │◀──────────────────│                    │                    │         │
│        │    (MCP server is the authorization server)                  │         │
│        │                    │                    │                    │         │
│   3. Authorization request to MCP server (PKCE)  │                    │         │
│        │───────────────────▶│                    │                    │         │
│        │                    │                    │                    │         │
│   4. MCP server redirects to CILogon             │                    │         │
│        │                    │───────────────────▶│                    │         │
│        │                    │                    │                    │         │
│   5. User authenticates (institution login)      │                    │         │
│        │                    │                    │◀───────────────────▶│         │
│        │                    │                    │     (browser)      │         │
│        │                    │                    │                    │         │
│   6. CILogon redirects to MCP server callback    │                    │         │
│        │                    │◀──────────────────│                    │         │
│        │                    │    (auth code)     │                    │         │
│        │                    │                    │                    │         │
│   7. MCP server exchanges code for CILogon token │                    │         │
│        │                    │───────────────────▶│                    │         │
│        │                    │◀──────────────────│                    │         │
│        │                    │  (access_token, user info)              │         │
│        │                    │                    │                    │         │
│   8. MCP server redirects to MCP client callback │                    │         │
│        │◀──────────────────│                    │                    │         │
│        │   (MCP auth code)  │                    │                    │         │
│        │                    │                    │                    │         │
│   9. Client exchanges code for MCP token         │                    │         │
│        │───────────────────▶│                    │                    │         │
│        │◀──────────────────│                    │                    │         │
│        │  (MCP access_token)│                    │                    │         │
│        │                    │                    │                    │         │
│  10. MCP tool calls with Bearer token            │                    │         │
│        │───────────────────▶│                    │                    │         │
│        │                    │                    │                    │         │
│  11. Call backend with service token + X-Acting-User                  │         │
│        │                    │────────────────────────────────────────▶│         │
│        │                    │◀───────────────────────────────────────│         │
│        │                    │                    │                    │         │
│  12. Response to client     │                    │                    │         │
│        │◀──────────────────│                    │                    │         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Why the Proxy Pattern?

MCP clients (Claude, ChatGPT, etc.) each have different OAuth callback URLs. CILogon requires exact redirect URI matches with no wildcard support. The proxy pattern solves this:

| Approach | Problem |
|----------|---------|
| Register every MCP client's callback with CILogon | Impractical — new clients require CILogon registration changes |
| Use wildcards | CILogon doesn't support redirect URI wildcards |
| **MCP server proxies OAuth** | **Single redirect URI; MCP server handles all clients** |

With the proxy pattern:

- CILogon sees only one client: the ACCESS MCP server
- MCP clients see the ACCESS MCP server as their authorization server
- Adding support for new MCP clients requires no CILogon changes

This is the standard pattern used by Cloudflare, Stytch, and other production MCP deployments.

---

## Infrastructure

### Current State

- 10 MCP servers with 24 tools deployed on Linode
- Streamable HTTP transport

### Changes Required

| Component | Change |
|-----------|--------|
| MCP server | Add OAuth authorization server endpoints |
| MCP server | Add OAuth proxy logic for CILogon |
| MCP server | Add token generation/validation |
| CILogon | Register single OAuth client |
| DNS | Point `mcp.access-ci.org` to Linode |

---

## OAuth Endpoints

The MCP server exposes these OAuth endpoints to MCP clients:

| Endpoint | Path | Purpose |
|----------|------|---------|
| Protected Resource Metadata | `/.well-known/oauth-protected-resource` | Advertises auth server location |
| Authorization Server Metadata | `/.well-known/oauth-authorization-server` | OAuth configuration for clients |
| Authorization | `/oauth/authorize` | Initiates auth flow, redirects to CILogon |
| Token | `/oauth/token` | Exchanges codes for tokens |
| Callback | `/oauth/callback` | Receives CILogon redirect |

### Protected Resource Metadata

**Endpoint**: `https://mcp.access-ci.org/.well-known/oauth-protected-resource`

Tells MCP clients where to find authorization server metadata.

### Authorization Server Metadata

**Endpoint**: `https://mcp.access-ci.org/.well-known/oauth-authorization-server`

Provides OAuth configuration:

- Authorization endpoint
- Token endpoint
- Supported scopes
- Supported grant types
- PKCE support (required)

---

## CILogon Configuration

### Single OAuth Client Registration

Register one OAuth client with CILogon:

| Setting | Value |
|---------|-------|
| Client Name | ACCESS-CI MCP Server |
| Callback URL | `https://mcp.access-ci.org/oauth/callback` |
| Scopes | `openid profile email org.cilogon.userinfo` |
| Client Type | Confidential |

Only one redirect URI needed — the MCP server's callback endpoint.

### CILogon Endpoints Used

| Endpoint | URL |
|----------|-----|
| Authorization | `https://cilogon.org/authorize` |
| Token | `https://cilogon.org/oauth2/token` |
| Userinfo | `https://cilogon.org/oauth2/userinfo` |
| JWKS | `https://cilogon.org/oauth2/certs` |

### User Identity

CILogon provides (with `org.cilogon.userinfo` scope):

| Claim | Description | Use |
|-------|-------------|-----|
| `eppn` | eduPersonPrincipalName | **ACCESS ID** (canonical user identifier) |
| `sub` | Unique user identifier | Token subject |
| `email` | Email address | Contact/display |
| `name` | Display name | UI display |
| `idp` | Identity provider used | Audit |

**Key mapping**: CILogon `eppn` = ACCESS ID. This is the canonical identifier used across all ACCESS systems.

---

## Token Strategy

The MCP server issues its own JWT tokens after validating CILogon authentication. This provides:

- Control over token lifetime and claims
- Ability to include ACCESS-specific claims
- Independent token revocation
- Consistent token format for all MCP tools

### MCP Token Claims

| Claim | Value | Description |
|-------|-------|-------------|
| `iss` | `https://mcp.access-ci.org` | Issuer |
| `sub` | User's ACCESS ID | Subject (= CILogon eppn) |
| `aud` | `https://mcp.access-ci.org` | Audience |
| `exp` | Issued + 1 hour | Expiration |
| `iat` | Current time | Issued at |
| `access_id` | User's ACCESS ID | Explicit ACCESS identifier |
| `name` | Display name | From CILogon |
| `email` | Email address | From CILogon |
| `scope` | Granted scopes | What user can do |

### Token Lifetime

| Token | Lifetime | Notes |
|-------|----------|-------|
| Access token | 1 hour | Balance security with usability |
| Refresh token | 7 days | For longer research sessions |

---

## Authorization

### Scopes

| Scope | Description | Access Level |
|-------|-------------|--------------|
| `openid` | OpenID Connect identity | Required |
| `profile` | Name, ACCESS ID | Basic |
| `email` | Email address | Basic |
| `access:read` | Query ACCESS data | All authenticated users |
| `access:write` | Create/edit content | Authorized roles only |

### Tool Authorization

| Tool Category | Required Scope |
|---------------|----------------|
| Software Discovery | `access:read` |
| Compute Resources | `access:read` |
| XDMoD Analytics | `access:read` |
| Allocations | `access:read` |
| Announcements (read) | `access:read` |
| Announcements (write) | `access:write` |
| Events (read) | `access:read` |
| Events (write) | `access:write` |

### Backend Authorization

Scopes control access at the MCP layer. Fine-grained authorization happens at each backend:

- MCP server checks: Does user have required scope?
- Backend checks: Does user have permission for this specific action?

See [Backend Integration Spec](./07-backend-integration-spec.md) for the backend authorization contract.

---

## Backend Integration

For all backend API calls, the MCP server uses the service account pattern:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         BACKEND CALL PATTERN                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   MCP Server                              Backend API                           │
│       │                                        │                                │
│       │   Authorization: Bearer {service_token}                                 │
│       │   X-Acting-User: jsmith@access-ci.org                                   │
│       │   X-Request-ID: 550e8400-e29b-...                                       │
│       │───────────────────────────────────────▶│                                │
│       │                                        │                                │
│       │                          Backend validates:                             │
│       │                          1. Service token (is this MCP server?)         │
│       │                          2. Acting user (who is this for?)              │
│       │                          3. Authorization (can they do this?)           │
│       │                                        │                                │
│       │◀───────────────────────────────────────│                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Headers Sent to Backends

| Header | Value | Purpose |
|--------|-------|---------|
| `Authorization` | `Bearer {service_token}` | Authenticates MCP server to backend |
| `X-Acting-User` | User's ACCESS ID | Identifies user for authorization/attribution |
| `X-Request-ID` | UUID | Correlation ID for tracing |

### Backend Responsibilities

Each backend API must:

1. Validate the service token (authenticate MCP server)
2. Parse `X-Acting-User` header
3. Look up user by ACCESS ID
4. Apply authorization rules
5. Attribute actions to the user
6. Log with `X-Request-ID` for tracing

Full specification: [Backend Integration Spec](./07-backend-integration-spec.md)

---

## Security Considerations

### OAuth Security

| Risk | Mitigation |
|------|------------|
| Auth code interception | PKCE required (S256) |
| Token theft | Short-lived tokens, HTTPS only |
| Open redirect | Validate redirect URIs against registered values |
| Token replay | Include `iat`, enforce expiration |

### MCP-Specific Security

| Risk | Mitigation |
|------|------------|
| Prompt injection | Validate/sanitize tool inputs |
| Unauthorized writes | Check scopes at MCP layer, permissions at backend |
| Cross-client token use | Validate `aud` claim matches MCP server |

### Audit Logging

Log for all operations:

- Timestamp
- Request ID
- User identifier (ACCESS ID)
- MCP client (from token if available)
- Tool invoked
- Backend called
- Success/failure

---

## Implementation Phases

### Phase 1: CILogon Registration

- [ ] Register OAuth client with CILogon
- [ ] Configure callback URL: `https://mcp.access-ci.org/oauth/callback`
- [ ] Request scopes: `openid profile email org.cilogon.userinfo`
- [ ] Store client credentials securely
- [ ] Test OAuth flow manually (browser + curl)

**Deliverable**: Working CILogon client credentials

### Phase 2: OAuth Authorization Server

- [ ] Implement `/.well-known/oauth-protected-resource`
- [ ] Implement `/.well-known/oauth-authorization-server`
- [ ] Implement `/oauth/authorize` (redirects to CILogon)
- [ ] Implement `/oauth/callback` (receives CILogon redirect)
- [ ] Implement `/oauth/token` (exchanges codes, issues tokens)
- [ ] Implement token generation (JWT signing)
- [ ] Implement PKCE validation

**Deliverable**: MCP server acts as OAuth authorization server

### Phase 3: Token Validation

- [ ] Add token validation middleware to MCP tools
- [ ] Validate JWT signature
- [ ] Check issuer, audience, expiration
- [ ] Extract `access_id` for backend calls
- [ ] Implement scope checking

**Deliverable**: MCP tools require valid tokens

### Phase 4: Backend Integration

- [ ] Implement service token management
- [ ] Implement `X-Acting-User` header injection
- [ ] Implement `X-Request-ID` generation
- [ ] Work with backend teams to implement contract
- [ ] Test end-to-end with Drupal (pilot backend)

**Deliverable**: Authenticated users can perform actions via backends

### Phase 5: Client Testing

- [ ] Test connection from Claude (claude.ai)
- [ ] Test connection from Claude Desktop
- [ ] Test connection from ChatGPT
- [ ] Test connection from VS Code/Cursor
- [ ] Document any client-specific issues

**Deliverable**: Verified working with major MCP clients

### Phase 6: Documentation and Launch

- [ ] Create user guide for connecting AI tools
- [ ] Document available tools
- [ ] Create troubleshooting guide
- [ ] Announce to ACCESS community

**Deliverable**: Researchers using ACCESS MCP

---

## Client Connection Guide

### Claude (claude.ai / Desktop / Mobile)

1. Settings → Connectors → Add custom connector
2. Enter: `https://mcp.access-ci.org/mcp`
3. Click Connect
4. Authenticate via CILogon (select your institution)
5. Tools now available in Claude

### ChatGPT

1. Settings → Connections → Add MCP Server
2. Enter: `https://mcp.access-ci.org/mcp`
3. Complete CILogon authentication
4. Tools available in ChatGPT

### VS Code / Cursor

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "access-ci": {
      "url": "https://mcp.access-ci.org/mcp",
      "type": "http"
    }
  }
}
```

Authenticate when prompted.

---

## Configuration Summary

### MCP Server

| Setting | Value |
|---------|-------|
| MCP Endpoint | `https://mcp.access-ci.org/mcp` |
| OAuth Metadata | `/.well-known/oauth-authorization-server` |
| Authorization | `/oauth/authorize` |
| Token | `/oauth/token` |
| Callback | `/oauth/callback` |

### CILogon Client

| Setting | Value |
|---------|-------|
| Callback URL | `https://mcp.access-ci.org/oauth/callback` |
| Scopes | `openid profile email org.cilogon.userinfo` |
| Client Type | Confidential |

### Backend Integration

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer {service_token}` |
| `X-Acting-User` | User's ACCESS ID |
| `X-Request-ID` | UUID for tracing |

See [Backend Integration Spec](./07-backend-integration-spec.md) for full contract.

---

## References

- [MCP Authorization Specification (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- [MCP Transport Specification (2025-03-26)](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports)
- [Cloudflare MCP Authorization](https://developers.cloudflare.com/agents/model-context-protocol/authorization/)
- [CILogon OIDC Documentation](https://www.cilogon.org/oidc)
- [RFC 6749 - OAuth 2.0](https://tools.ietf.org/html/rfc6749)
- [RFC 7636 - PKCE](https://tools.ietf.org/html/rfc7636)
- [RFC 9728 - Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728)
- [Claude Custom Connectors](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp)
