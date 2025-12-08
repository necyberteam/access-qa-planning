# Events Actions (Pilot)

> **Related**: [Agent Architecture](./01-agent-architecture.md)

## Overview

**Goal**: Enable AI agents to take actions on behalf of users - not just answer questions, but actually do things.

Currently, the QA Bot can only read data. This pilot extends it to perform authenticated operations, allowing users to say "Create a workshop for next Tuesday" and have the agent actually create the event.

Events management is the first use case, but the patterns established here apply to any future action tools.

### Why Events as the Pilot

| Factor | Events | Other Candidates |
|--------|--------|------------------|
| User demand | Users frequently create events | - |
| Risk level | Low - events are public, content-moderated | Higher for user/allocation data |
| API availability | Drupal Events API exists | Would need new APIs |
| Complexity | Moderate - good test case | - |

### Future Action Tools

Once patterns are proven:
- Affinity group membership management
- Support ticket creation
- Documentation feedback/suggestions
- Resource allocation requests

---

## Current State vs Target

**Current**:
- Events API (`/api/2.2/events`) is read-only, public
- Events created only via Drupal admin UI
- QA Bot receives user context but can't take actions

**Target**:
- Authenticated create/update/delete operations via MCP tools
- Users create events conversationally through QA Bot
- Events follow content moderation (draft → published)
- Full audit trail

---

## Architecture

```
User → QA Bot → n8n Orchestrator → Events MCP Server → Drupal Events API
                                          │
                                          ├── Server validates JWT
                                          └── Tool calls Drupal with service API key
```

### Key Principle: No Token Passthrough

Per MCP spec, the server MUST NOT forward user tokens to upstream APIs. Instead:

1. MCP Server validates user's JWT (audience = `mcp://events`)
2. MCP Server extracts user identity from JWT claims
3. When tool executes, it uses MCP server's own API key to call Drupal
4. User identity passed to Drupal via `X-Acting-User` header

---

## Authentication Flow

```
1. User logs in via CILogon → ACCESS IdP → Drupal session

2. Drupal generates JWT for QA Bot:
   {
     "sub": "12345",
     "access_id": "user@access-ci.org",
     "email": "user@example.edu",
     "name": "Jane Researcher",
     "roles": ["authenticated", "event_creator"],
     "exp": <1 hour>,
     "aud": "mcp://events",
     "iss": "https://support.access-ci.org"
   }

3. QA Bot passes JWT to n8n → MCP Server
   - MCP Server validates JWT signature and audience
   - Extracts user claims for authorization decisions

4. When tool executes (e.g., create_event), it calls Drupal:
   Headers:
     X-API-Key: <mcp_server_service_key>
     X-Acting-User: user@access-ci.org
```

---

## MCP Tools

### create_event (requires auth)

Create a new ACCESS event. Created in draft status.

**Required**: title, event_type, location, start_date
**Optional**: description, end_date, registration_url, skill_level, tags

**event_type**: Training, Conference, Office Hours, Other
**skill_level**: Beginner, Intermediate, Advanced

### update_event (requires auth)

Update an existing event. User must own the event or be admin.

**Required**: event_id
**Optional**: Any field from create_event

### delete_event (requires auth)

Delete an event. User must own the event or be admin.

**Required**: event_id

### get_my_events (requires auth)

List events created by the authenticated user.

**Optional**: status (draft, published, all), limit

---

## Authorization Matrix

| Action | Any Auth User | Event Owner | Admin |
|--------|---------------|-------------|-------|
| Create event (draft) | ✓ | - | ✓ |
| View own events | ✓ | - | ✓ |
| Update own event | - | ✓ | ✓ |
| Delete own event | - | ✓ | ✓ |
| Publish event | ✗ | ✗* | ✓ |
| Update any event | ✗ | ✗ | ✓ |

*Users with `publish` permission can publish their own events

---

## Content Moderation

Events created via API always start as **draft**:

```
API CREATE → DRAFT → (Drupal UI review) → PUBLISHED
```

This ensures:
- Staff can review before public visibility
- No spam/inappropriate content published automatically
- Same workflow as manual event creation

---

## Conversational Flow

When user says "I want to create a workshop":

1. **Detect intent**: Query classifier recognizes event creation
2. **Check auth**: If not logged in, prompt to login
3. **Collect info**: Conversationally gather required fields
   - "What would you like to call this event?"
   - "What type of event is this?" (show options)
   - "When will it take place?"
   - "Where will it be held? (or 'Virtual')"
4. **Confirm**: Show summary, ask for confirmation
5. **Create**: Call MCP tool with collected data
6. **Response**: "Created! It's in draft status. Review here: [edit_url]"

### Progressive Confirmation

Risk-based confirmation levels:

| Action | Risk | Confirmation Required |
|--------|------|----------------------|
| Create event (draft) | Low | Summary + "Does this look right?" |
| Update event details | Low | Show diff + "Update these fields?" |
| Delete draft event | Medium | "Are you sure you want to delete [title]?" |
| Delete published event | High | "This will remove [title] from the public calendar. Type 'delete' to confirm." |

**Undo Support** (where possible):
- Deleted drafts recoverable for 24 hours
- Published events move to "archived" before permanent deletion

---

## Drupal API Endpoints

New module: `access_events_api`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/events/manage` | Create event |
| PATCH | `/api/events/manage/{id}` | Update event |
| DELETE | `/api/events/manage/{id}` | Delete event |
| GET | `/api/events/manage/mine` | List user's events |

Authentication:
- `X-API-Key`: Service key for MCP server
- `X-Acting-User`: ACCESS ID from validated JWT

---

## Security

### Token Security

| Concern | Mitigation |
|---------|------------|
| Token theft | 1-hour expiry, HTTPS only |
| Token replay | Include `iat` claim |
| Privilege escalation | Roles in token, server-side validation |
| API key exposure | Stored in Drupal key module, not in client |

### Audit Logging

All API operations logged:
- Timestamp
- User ID (from token)
- Action performed
- Event ID affected
- IP address
- Success/failure

---

## Implementation Phases

### Phase 1: Drupal Events API
- [ ] Create `access_events_api` module
- [ ] Implement authentication middleware
- [ ] Implement CRUD endpoints
- [ ] Tests

### Phase 2: Events MCP Server
- [ ] Add authenticated tools to events server
- [ ] Implement JWT validation
- [ ] Add Drupal API client
- [ ] Tests

### Phase 3: n8n Integration
- [ ] Update query classifier for event management intent
- [ ] Create conversational flow for event creation
- [ ] Connect to MCP server

### Phase 4: QA Bot Integration
- [ ] Add token generation to Drupal
- [ ] Update QA Bot to pass token
- [ ] End-to-end testing
- [ ] Security review

---

## Reusable Patterns for Future Tools

This pilot establishes patterns for any authenticated action tool:

1. **JWT delegation**: Web app creates scoped JWT with user claims
2. **Audience separation**: Each tool domain has own audience
3. **Service key pattern**: MCP server authenticates to upstream with own key
4. **Acting-user header**: Pass user identity for audit/authorization
5. **Content moderation**: Actions create drafts, not public content
6. **Conversational collection**: Gather required info through dialogue

---

## Open Questions

### Technical
1. **Key management**: Drupal key module or external vault?
2. **Token refresh**: Require page reload or implement refresh?
3. **Rate limiting**: What limits per user?

### Policy
4. **Who can create events?**: All authenticated users or specific roles?
5. **Auto-publish**: Should any users publish directly via API?

### UX
6. **Error messages**: How detailed?

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Event creation success rate | ≥95% |
| Time to create event (conversation) | <2 minutes |
| User satisfaction | ≥80% positive |
| Security incidents | 0 |
| API latency (P95) | <2s |
