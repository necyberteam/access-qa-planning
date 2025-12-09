# MCP Action Tools

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [MCP Authentication](./06-mcp-authentication.md)

## Overview

**Goal**: Enable AI agents to take actions on behalf of users - not just answer questions, but actually do things.

Currently, the QA Bot can only read data. This initiative extends it to perform authenticated operations, allowing users to create content conversationally through the QA Bot.

### Phased Approach

| Phase | Content Type | Complexity | Status |
|-------|--------------|------------|--------|
| **Phase 1** | Announcements | Lower - simple fields, standard node | **Active** |
| **Phase 2** | Events | Higher - dates, locations, recurring_events module | Planned |

### Why Announcements First

| Factor | Announcements | Events |
|--------|---------------|--------|
| Content type | `access_news` (standard node) | `eventseries` (recurring_events module) |
| Field complexity | Simple - title, body, tags | Complex - dates, locations, registration |
| Validation logic | Coordinator check for affinity groups | Similar + date handling |
| Development effort | Lower | Higher |

Starting with Announcements lets us establish patterns with a simpler content type before tackling the more complex Events system.

### Future Action Tools

Once patterns are proven:
- Affinity group membership management
- Support ticket creation
- Documentation feedback/suggestions
- Resource allocation requests

---

## Detailed Documentation

| Document | Purpose |
|----------|---------|
| [06-mcp-authentication.md](./06-mcp-authentication.md) | Authentication architecture (JWT, API keys, flow diagrams) |
| [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md) | Phase 1: Drupal developer spec for Announcements API |

---

## Architecture Summary

```
User → QA Bot → n8n Orchestrator → MCP Server → Drupal JSON:API
                                       │              │
                                       │              ├── Key Auth (service account)
                                       │              └── access_mcp_author hook
                                       │
                                       ├── Validates user JWT
                                       └── Calls JSON:API as mcp_service + X-Acting-User header
```

### Key Principles

1. **Service Account Pattern**: MCP server authenticates as `mcp_service` Drupal user
2. **Standard JSON:API**: Uses Drupal's built-in JSON:API for CRUD operations
3. **Key Auth Module**: Service account authenticated via `api-key` header
4. **Author Swap Hook**: `access_mcp_author` module changes node author to acting user
5. **Draft-First**: All content created via API starts unpublished

→ *Full details: [06-mcp-authentication.md](./06-mcp-authentication.md)*

---

## Phase 1: Announcements

### Current State
- Announcements API (`/api/2.2/announcements`) is read-only, public
- Announcements created only via Drupal admin UI
- QA Bot receives user context but can't take actions

### Target State
- Authenticated create/update/delete operations via MCP tools
- Users create announcements conversationally through QA Bot
- Announcements follow content moderation (draft → published)
- Full audit trail

### Key Constraints (from existing Drupal code)

| Constraint | Detail |
|------------|--------|
| Body format | `basic_html` only |
| Tags | Must be existing taxonomy terms (1-6 required) |
| Affinity groups | User must be coordinator of the group |

### JSON:API Endpoints

Uses standard Drupal JSON:API (no custom endpoints needed for CRUD):

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/jsonapi/node/access_news` | POST | Create announcement (draft) |
| `/jsonapi/node/access_news/{uuid}` | PATCH | Update announcement |
| `/jsonapi/node/access_news/{uuid}` | DELETE | Delete announcement |
| `/jsonapi/node/access_news?filter[uid.id]=xxx` | GET | List user's announcements |
| `/jsonapi/taxonomy_term/tags` | GET | List available tags |
| `/jsonapi/node/affinity_group` | GET | List affinity groups |

### Implementation Checklist

- [ ] Install Key Auth module
- [ ] Create `mcp_service` user account and role
- [ ] Enable JSON:API writes
- [ ] Create `access_mcp_author` module (author swap + validation)
- [ ] Tests
- [ ] MCP server integration

→ *Full spec: [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md)*

---

## Phase 2: Events

### Overview

Events are more complex than Announcements due to:
- `eventseries` content type (recurring_events module)
- Date/time handling with recurrence
- Location management
- Registration URLs and capacity

### Drupal Module: `access_events_api`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/events/manage` | POST | Create event (draft) |
| `/api/events/manage/{id}` | PATCH | Update event |
| `/api/events/manage/{id}` | DELETE | Delete event |
| `/api/events/manage/mine` | GET | List user's events |

### MCP Tools

| Tool | Auth | Description |
|------|------|-------------|
| `create_event` | Required | Create new event (draft status) |
| `update_event` | Required | Update existing event (owner or admin) |
| `delete_event` | Required | Delete event (owner or admin) |
| `get_my_events` | Required | List user's events |

### Authorization Matrix

| Action | Any Auth User | Owner | Admin |
|--------|---------------|-------|-------|
| Create event (draft) | ✅ | - | ✅ |
| View own events | ✅ | - | ✅ |
| Update event | ❌ | ✅ | ✅ |
| Delete event | ❌ | ✅ | ✅ |
| Publish event | ❌ | ❌ | ✅ |

---

## Content Moderation (Both Phases)

All content created via API starts as **draft**:

```
API CREATE → DRAFT → (Staff review in Drupal UI) → PUBLISHED
```

This ensures:
- Staff can review before public visibility
- No spam/inappropriate content published automatically
- Same workflow as manual content creation

---

## Conversational Flow Pattern

When user expresses intent to create content:

1. **Detect intent**: Query classifier recognizes creation request
2. **Check auth**: If not logged in, prompt to login
3. **Collect info**: Conversationally gather required fields
4. **Validate**: Check constraints (tags exist, coordinator permission, etc.)
5. **Confirm**: Show summary, ask for confirmation
6. **Create**: Call MCP tool with collected data
7. **Response**: "Created! It's in draft status. Review here: [edit_url]"

### Progressive Confirmation

| Action | Risk | Confirmation |
|--------|------|--------------|
| Create (draft) | Low | Summary + "Does this look right?" |
| Update | Low | Show diff + "Update these fields?" |
| Delete draft | Medium | "Are you sure you want to delete [title]?" |
| Delete published | High | Explicit confirmation required |

---

## Reusable Patterns

These patterns apply to any authenticated action tool:

| Pattern | Description |
|---------|-------------|
| Service account | `mcp_service` Drupal user for API access |
| Key Auth | Standard module for API key authentication |
| JSON:API | Drupal core for CRUD operations |
| Acting-user header | `X-Acting-User` header identifies real user |
| Author swap hook | `access_mcp_author` changes owner to acting user |
| JWT for user identity | MCP server validates user JWT, extracts access_id |
| Draft-first | Actions create drafts, not public content |

---

## Security

→ *Full details: [06-mcp-authentication.md](./06-mcp-authentication.md)*

### Summary

| Concern | Mitigation |
|---------|------------|
| Token theft | 1-hour expiry, HTTPS only |
| Token replay | Include `iat` claim |
| Privilege escalation | Roles in token, server-side validation |
| API key exposure | Stored in Drupal Key module, not in client |

### Audit Logging

All API operations logged:
- Timestamp
- User (from `X-Acting-User`)
- Action performed
- Entity affected
- Success/failure

---

## Open Questions

| Area | Question |
|------|----------|
| Technical | Token refresh: require page reload or implement refresh? |
| Technical | Rate limiting: what limits per user per hour? |
| Policy | Auto-publish: should any users publish directly via API? |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Content creation success rate | ≥95% |
| Time to create (conversation) | <2 minutes |
| User satisfaction | ≥80% positive |
| Security incidents | 0 |
| API latency (P95) | <2s |
