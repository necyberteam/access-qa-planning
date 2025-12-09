# Drupal Announcements API Specification

> **For**: Drupal Developer
> **Module**: `access_announcements_api`
> **Priority**: Pilot for MCP Action Tools
> **Related**: [Events Actions Plan](./05-events-actions.md) (future Phase 2)

## Overview

Create a new Drupal module that exposes authenticated API endpoints for managing Announcements (`access_news` content type). This enables the AI QA Bot to create, update, and delete announcements on behalf of authenticated users.

### Why Announcements First (Before Events)

| Factor | Announcements | Events |
|--------|---------------|--------|
| **Field complexity** | Simple - title, body, optional metadata | Complex - dates, locations, registration, recurrence |
| **Content type** | `access_news` (standard node) | `eventseries` (recurring_events module) |
| **Risk level** | Low - internal communications | Same |
| **Development effort** | Lower | Higher |

---

## Content Type Reference

**Machine name**: `access_news`
**Human name**: Announcement

### Fields

| Field | Machine Name | Type | Required | Notes |
|-------|--------------|------|----------|-------|
| Title | `title` | string | **Yes** | Node title |
| Body | `body` | text_with_summary | No | **basic_html format only** (see constraints) |
| Published Date | `field_published_date` | datetime | No | When to display |
| Affiliation | `field_affiliation` | list_string | No | `ACCESS Collaboration` or `Community` |
| Affinity Group Node | `field_affinity_group_node` | entity_reference | No | References affinity_group nodes (**coordinator permission required**) |
| Affinity Group | `field_affinity_group` | entity_reference | No | Auto-set from node (see presave hook) |
| Tags | `field_tags` | entity_reference | No | **Must be existing** `tags` taxonomy terms (1-6 required) |
| Image | `field_image` | image | No | Optional (defer to Phase 2) |
| External Link | `field_news_external_link` | link | No | Link to external source |

### Important Constraints from Existing Code

The following validation rules exist in `access_news.module` and must be enforced by the API:

1. **Body format**: Only `basic_html` format is allowed (configured in field settings)

2. **Tags**:
   - Must be existing taxonomy terms from the `tags` vocabulary
   - Minimum 1 tag required, maximum 6 tags
   - Tags are NOT auto-created; unknown tags should be rejected

3. **Affinity Group assignment** (from `access_news_validate()`):
   - Users can only assign an Affinity Group if they are a **Coordinator** of that group
   - Coordinators are listed in the `field_coordinator` field on the affinity_group node
   - Exception: users with `administrator` or `news_pm` roles can assign any group
   - The `field_affinity_group` taxonomy term is auto-populated from `field_affinity_group_node` via the `access_news_node_presave()` hook

4. **Affinity Group taxonomy**: When assigning an affinity group node, the corresponding taxonomy term must exist (validated in existing code)

---

## Authentication Pattern

All endpoints use a two-header authentication approach:

| Header | Value | Purpose |
|--------|-------|---------|
| `X-API-Key` | Service key | Validates the calling service (MCP server) |
| `X-Acting-User` | ACCESS ID (e.g., `user@access-ci.org`) | Identifies who is performing the action |

**Flow:**
1. MCP server validates user's JWT token
2. MCP server extracts `access_id` from validated token
3. MCP server calls Drupal API with service key + acting user headers
4. Drupal looks up user by `field_access_id` and performs action as that user

**Security principle**: User tokens are never passed through to Drupal. The MCP server acts as a trusted intermediary.

---

## API Endpoints

### Base Path: `/api/announcements/manage`

### 1. Create Announcement

**POST** `/api/announcements/manage`

Creates a new announcement in **draft** status (unpublished).

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `title` | string | **Yes** | Announcement title |
| `body` | string | No | HTML content (**basic_html format enforced**) |
| `published_date` | date (YYYY-MM-DD) | No | Display date |
| `affiliation` | string | No | `ACCESS Collaboration` or `Community` |
| `affinity_group` | string | No | Group ID or name (**user must be coordinator**) |
| `tags` | array of strings | **Yes** | 1-6 tag names (**must be existing taxonomy terms**) |

**Success Response (201):**
- `success`: true
- `announcement_id`: integer
- `status`: "draft"
- `message`: User-friendly confirmation
- `edit_url`: URL to edit in Drupal
- `view_url`: URL to view (once published)

---

### 2. Update Announcement

**PATCH** `/api/announcements/manage/{id}`

Updates an existing announcement. User must be the owner or an admin.

All fields from Create are accepted; only include fields to update.

**Success Response (200):**
- `success`: true
- `announcement_id`: integer
- `status`: current status
- `message`: User-friendly confirmation
- `edit_url`: URL to edit

**Error Response (403):** User doesn't own the announcement

---

### 3. Delete Announcement

**DELETE** `/api/announcements/manage/{id}`

Deletes an announcement. User must be the owner or an admin.

**Success Response (200):**
- `success`: true
- `message`: Confirmation

---

### 4. List User's Announcements

**GET** `/api/announcements/manage/mine`

Returns announcements created by the authenticated user.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `status` | `all` | Filter: `draft`, `published`, or `all` |
| `limit` | 20 | Max 100 |
| `offset` | 0 | Pagination |

**Response (200):**
- `total`: Total count matching filters
- `items`: Array of announcement summaries with id, title, status, created, edit_url

---

## Authorization Matrix

| Action | Any Auth User | Owner | Admin |
|--------|---------------|-------|-------|
| Create announcement (draft) | ✅ | - | ✅ |
| List own announcements | ✅ | - | ✅ |
| View own announcement | ✅ | - | ✅ |
| Update announcement | ❌ | ✅ | ✅ |
| Delete announcement | ❌ | ✅ | ✅ |
| Publish announcement | ❌ | ❌ | ✅ |

**Key constraint**: Publishing is NOT exposed via this API. All announcements created via API start as drafts and must be published through the Drupal admin UI.

---

## Error Responses

All error responses follow this format:

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Always `false` |
| `error` | string | Error code (e.g., `validation_error`, `not_found`, `forbidden`) |
| `message` | string | Human-readable message |
| `field` | string | (optional) Field that caused validation error |

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 201 | Created successfully |
| 200 | Operation successful |
| 400 | Bad request / validation error |
| 401 | Missing or invalid API key |
| 403 | User doesn't have permission |
| 404 | Announcement not found |
| 500 | Server error |

---

## Module Requirements

### Dependencies
- `node` (core)
- `key` module (for API key storage - already installed)

---

## Existing Drupal Patterns to Follow

The codebase already has patterns for APIs and key management. Use these existing approaches:

### 1. Key Module for API Key Storage (Existing)

| Reference | Location |
|-----------|----------|
| `SimpleListsApi` | `access_affinitygroup/src/Plugin/SimpleListsApi.php` |
| `ConstantContactApi` | `access_affinitygroup/src/Plugin/ConstantContactApi.php` |
| `AiReferenceGenerator` | `access_llm/src/AiReferenceGenerator.php` |

Use `key.repository` service to retrieve key values. For this module:
- Key ID: `mcp_service_key`
- Set via: `drush key:set mcp_service_key "your-key"`
- Shared across all MCP servers (see [06-mcp-authentication.md](./06-mcp-authentication.md))

### 2. Custom Controller with JsonResponse (Existing)

| Reference | Location | Purpose |
|-----------|----------|---------|
| `JsonApiAccessController` | `access/src/Controller/JsonApiAccessController.php` | Organization autocomplete |
| `JsonApiCCController` | `campuschampions/src/Controller/JsonApiCCController.php` | Carnegie code lookup |

Follow this pattern: custom routing → controller method → `JsonResponse`

### 3. Views REST Export (Existing, Read-Only)

The existing `/api/2.2/announcements` endpoint uses Views REST export (`views.view.access_announcements.yml`). Good for read-only queries but not suitable for write operations.

### 4. What's New (Build as Shared Service)

The following are **new** and should be built as a **shared service** for reuse by future APIs (Events, etc.):

| Component | Description |
|-----------|-------------|
| `X-API-Key` validation | Validate service key from Key module |
| `X-Acting-User` handling | Look up Drupal user by `field_access_id` |
| Permission checking | Verify user can perform action |

**Recommendation**: Create `McpApiAuthenticator` service in a base module (`access_mcp_auth`) that API modules can depend on.

### 5. Logging (Existing)

Use Drupal's logger service (`\Drupal::logger('module_name')`) - same pattern used throughout codebase.

Log: action performed, user (from `X-Acting-User`), entity affected, success/failure

---

## Testing Requirements

### Test Cases

**Authentication:**
1. Create without API key - Returns 401
2. Create without X-Acting-User - Returns 401
3. Create with unknown user - Returns 401

**Basic CRUD:**
4. Create with valid credentials - Returns 201, creates draft
5. Update own announcement - Returns 200
6. Update someone else's announcement - Returns 403
7. Delete own announcement - Returns 200
8. Delete someone else's announcement - Returns 403
9. List my announcements - Returns only current user's
10. Invalid announcement ID - Returns 404

**Tag Validation:**
11. Create with no tags - Returns 400 (minimum 1 required)
12. Create with 7+ tags - Returns 400 (maximum 6 allowed)
13. Create with non-existent tag - Returns 400 with helpful error
14. Create with valid existing tags - Returns 201

**Affinity Group Validation:**
15. Assign affinity group as non-coordinator - Returns 403
16. Assign affinity group as coordinator - Returns 201
17. Assign affinity group as admin - Returns 201 (bypasses coordinator check)
18. Assign affinity group as news_pm role - Returns 201 (bypasses coordinator check)

**Body Format:**
19. Create with valid basic_html - Returns 201
20. Create with disallowed HTML tags - Body sanitized or rejected

### Manual Testing

Provide cURL examples in module README for testing each endpoint.

---

## Deployment Checklist

1. [ ] Module installed and enabled
2. [ ] API key set via Key module
3. [ ] Test user exists with `field_access_id` populated
4. [ ] All endpoints tested via cURL
5. [ ] Logging verified
6. [ ] Deployed to staging
7. [ ] MCP server configured with API key
8. [ ] End-to-end test with MCP server

---

## Future Considerations (Phase 2)

- **Image uploads**: Support for announcement images
- **Scheduled publishing**: Set future publish date via API
- **Bulk operations**: Create/update multiple announcements
- **Webhook notifications**: Notify MCP server when announcements are published

---

## Resolved Design Decisions

Based on existing Drupal validation logic in `access_news.module`:

| Question | Decision | Rationale |
|----------|----------|-----------|
| Tag handling | Reject unknown tags | Existing form validation requires existing taxonomy terms |
| Body format | basic_html only | Field config restricts to `basic_html` format |
| Affinity group permissions | Coordinator only | Existing `access_news_validate()` enforces this |
| Tag count | 1-6 required | Existing `access_news_tags_validate_element()` enforces this |

## Open Questions

1. **Rate limiting**: What limits per user per hour?

---

## Supporting MCP Tools / API Endpoints

Because tags must be existing taxonomy terms and affinity groups require coordinator permission, the AI agent needs ways to discover valid options. Consider these supporting endpoints:

### Option A: Add to Announcements API Module

#### GET `/api/announcements/manage/tags`

Returns available tags from the `tags` vocabulary.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `search` | - | Filter by name prefix |
| `limit` | 50 | Max results |

**Response:**
```json
{
  "items": [
    {"id": 123, "name": "gpu"},
    {"id": 124, "name": "hpc"}
  ]
}
```

#### GET `/api/announcements/manage/my-affinity-groups`

Returns affinity groups the authenticated user can post to (where they are coordinator, or if admin/news_pm).

**Response:**
```json
{
  "items": [
    {"id": 456, "name": "AI/CI", "group_id": "aici.access-ci.org"},
    {"id": 457, "name": "Campus Champions", "group_id": "champions.access-ci.org"}
  ]
}
```

### Option B: Separate MCP Tools (Read-Only)

Add tools to the existing announcements MCP server:

- `list_tags` - Returns available tags (no auth required, public data)
- `list_my_affinity_groups` - Returns groups user coordinates (requires auth)

### Recommendation

**Option A** is simpler for the pilot - keep everything in one module. The MCP server can call these endpoints to populate the AI's context before asking users about tags/groups.

For tag selection specifically, the existing form has AI-powered tag suggestion (using `access_llm.ai_references_generator`). Consider whether the MCP flow should:
1. Accept user-provided tags and validate them, OR
2. Suggest tags based on body content (matching existing UX)

---

## Implementation Notes

### Reusing Existing Validation Logic

The validation logic in `access_news.module` should ideally be reused rather than duplicated:

- `access_news_validate()` - Coordinator permission check
- `access_news_tags_validate_element()` - Tag count validation
- `update_affinity_group()` - Auto-populate taxonomy from node

**Options:**
1. **Extract to service**: Refactor validation into a service class that both the form and API can use
2. **Call existing functions**: Have API controller call the existing functions directly
3. **Duplicate with care**: Implement in API module but document the dependency

Option 1 is cleanest but requires touching existing code. Option 2 may work if functions are accessible. Option 3 is fastest but creates maintenance burden.

---

## Related Documents

- [Events Actions Plan](./05-events-actions.md) - Phase 2: Apply same pattern to Events
- [Agent Architecture](./01-agent-architecture.md) - Overall system design
- [n8n Events Management Plan](../n8n/EVENTS-MANAGEMENT-PLAN.md) - Detailed MCP/n8n integration (for Events, adaptable)
