# Drupal Announcements API Specification

> **For**: Drupal Developer
> **Module**: `access_mcp_author`
> **Priority**: Pilot for MCP Action Tools
> **Related**: [MCP Action Tools](./05-events-actions.md) | [MCP Authentication](./06-mcp-authentication.md)

## Overview

Enable the AI QA Bot to create, update, and delete announcements on behalf of authenticated users using **Drupal's built-in JSON:API** with a service account pattern.

### Approach

Instead of building custom API endpoints, we:
1. Use standard JSON:API with the [Key Auth](https://www.drupal.org/project/key_auth) module
2. Create a service account (`mcp_service`) that the MCP server authenticates as
3. Build a small module (`access_mcp_author`) that changes the content author to the acting user

### Why This Approach

| Factor | Custom Endpoints | JSON:API + Service Account |
|--------|------------------|---------------------------|
| API maintenance | Must maintain custom code | Uses Drupal core |
| Authentication | Custom implementation | Key Auth module (standard) |
| Validation | Must implement | Built-in JSON:API validation |
| Custom code needed | Full CRUD endpoints | Small hook module only |
| Request format | Simple JSON | JSON:API format (verbose but standard) |

---

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ MCP Server  │─────▶│  Key Auth   │─────▶│  JSON:API   │─────▶│    Node     │
│             │      │  (Drupal)   │      │  (Drupal)   │      │   Created   │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────┘
       │                   │                    │                     │
       │ api-key: xxx      │ Authenticates as   │                     │
       │ X-Acting-User:    │ mcp_service user   │                     │
       │ jsmith@access-ci  │                    │                     │
       │                   │                    │ hook_node_presave   │
       │                   │                    │ changes uid to      │
       │                   │                    │ acting user         │
```

---

## Setup Requirements

### 1. Install Key Auth Module

```bash
composer require drupal/key_auth
drush en key_auth
```

### 2. Create Service Account

Create a Drupal user for the MCP server:

| Field | Value |
|-------|-------|
| Username | `mcp_service` |
| Email | `mcp-service@access-ci.org` |
| Status | Active |
| Roles | Custom role with permissions (see below) |

### 3. Configure Key Auth

1. Go to `/admin/config/system/key-auth`
2. Enable **Header** detection method
3. Set header name: `api-key` (or your preference)
4. Generate a key for the `mcp_service` user

### 4. Service Account Permissions

Create a role `mcp_service_role` with:

| Permission | Purpose |
|------------|---------|
| `access content` | View content |
| `create access_news content` | Create announcements |
| `edit any access_news content` | Edit announcements (author changed in hook) |
| `delete any access_news content` | Delete announcements (validated in hook) |
| `use jsonapi` | Access JSON:API endpoints |

**Note**: The actual authorization (can this user create/edit/delete?) is validated in the hook based on the acting user, not the service account.

---

## Module: `access_mcp_author`

A small module that:
1. Detects requests from the `mcp_service` account
2. Reads the `X-Acting-User` header
3. Changes node ownership to the acting user
4. Validates the acting user has permission for the action

### Module Structure

```
web/modules/custom/access_mcp_author/
├── access_mcp_author.info.yml
├── access_mcp_author.module
└── src/
    └── EventSubscriber/
        └── McpAuthorSubscriber.php  (optional: for validation)
```

### Key Functionality

**hook_node_presave**: For create/update operations
- Verify request is from `mcp_service` user
- Read `X-Acting-User` header
- Look up Drupal user by `field_access_id`
- Validate acting user's permissions (coordinator check, etc.)
- Set node owner to acting user

**hook_node_predelete**: For delete operations
- Same validation pattern
- Verify acting user owns the node or is admin

### Validation Logic to Include

Reuse existing validation from `access_news.module`:

| Validation | Source | Apply To |
|------------|--------|----------|
| Tag count (1-6) | `access_news_tags_validate_element()` | Create, Update |
| Tags must exist | Field validation | Create, Update |
| Coordinator permission | `access_news_validate()` | Create, Update (when affinity group set) |
| Owner check | New | Update, Delete |

---

## JSON:API Endpoints

With JSON:API enabled for write operations (`read_only: false`), these endpoints become available:

### Create Announcement

**POST** `/jsonapi/node/access_news`

```
api-key: <mcp_service_key>
X-Acting-User: jsmith@access-ci.org
Content-Type: application/vnd.api+json
```

```json
{
  "data": {
    "type": "node--access_news",
    "attributes": {
      "title": "New GPU Resources Available",
      "status": false,
      "body": {
        "value": "<p>We're pleased to announce...</p>",
        "format": "basic_html"
      },
      "field_published_date": "2025-01-15",
      "field_affiliation": "ACCESS Collaboration"
    },
    "relationships": {
      "field_tags": {
        "data": [
          { "type": "taxonomy_term--tags", "id": "uuid-of-tag-1" },
          { "type": "taxonomy_term--tags", "id": "uuid-of-tag-2" }
        ]
      },
      "field_affinity_group_node": {
        "data": { "type": "node--affinity_group", "id": "uuid-of-group" }
      }
    }
  }
}
```

**Notes:**
- `status: false` creates as draft (unpublished)
- Relationships require UUIDs (MCP server fetches these via JSON:API reads)

### Update Announcement

**PATCH** `/jsonapi/node/access_news/{uuid}`

Same format as create, but include only fields to update.

### Delete Announcement

**DELETE** `/jsonapi/node/access_news/{uuid}`

```
api-key: <mcp_service_key>
X-Acting-User: jsmith@access-ci.org
```

### Read Operations (for MCP Server)

These are already available (JSON:API is read-enabled):

| Endpoint | Purpose |
|----------|---------|
| `GET /jsonapi/taxonomy_term/tags` | List available tags (with UUIDs) |
| `GET /jsonapi/node/affinity_group` | List affinity groups |
| `GET /jsonapi/node/access_news?filter[uid.id]=xxx` | List user's announcements |

---

## Content Type Reference

**Machine name**: `access_news`

### Fields

| Field | Machine Name | Type | Required | Notes |
|-------|--------------|------|----------|-------|
| Title | `title` | string | **Yes** | Node title |
| Body | `body` | text_with_summary | No | **basic_html format only** |
| Published Date | `field_published_date` | datetime | No | When to display |
| Affiliation | `field_affiliation` | list_string | No | `ACCESS Collaboration` or `Community` |
| Affinity Group Node | `field_affinity_group_node` | entity_reference | No | **Coordinator permission required** |
| Affinity Group | `field_affinity_group` | entity_reference | No | Auto-set from node |
| Tags | `field_tags` | entity_reference | **Yes** | 1-6 existing taxonomy terms |

### Validation Constraints

| Constraint | Enforced By |
|------------|-------------|
| Body format = basic_html | Field configuration |
| Tags: 1-6 required, must exist | `access_mcp_author` hook |
| Affinity group: coordinator only | `access_mcp_author` hook |
| Author = acting user | `access_mcp_author` hook |

---

## Authorization Matrix

| Action | Acting User Requirement |
|--------|------------------------|
| Create announcement | Any authenticated user (becomes owner) |
| Update announcement | Must be owner OR admin |
| Delete announcement | Must be owner OR admin |
| Assign affinity group | Must be coordinator of that group OR admin/news_pm |
| Publish announcement | NOT via API (staff only in Drupal UI) |

---

## Configuration Changes

### Enable JSON:API Writes

Update `jsonapi.settings.yml`:

```yaml
read_only: false  # Changed from true
```

**Security consideration**: This enables writes for ALL content types via JSON:API. The `access_mcp_author` module should validate that only authorized operations succeed.

Alternative: Use [JSON:API Resources](https://www.drupal.org/project/jsonapi_resources) module to create custom endpoints with controlled access.

---

## Implementation Checklist

### Setup
- [ ] Install and configure Key Auth module
- [ ] Create `mcp_service` user account
- [ ] Create `mcp_service_role` with appropriate permissions
- [ ] Generate API key for service account
- [ ] Enable JSON:API writes (or configure JSON:API Resources)

### Module Development
- [ ] Create `access_mcp_author` module
- [ ] Implement `hook_node_presave()` for author swap
- [ ] Implement `hook_node_predelete()` for delete validation
- [ ] Add tag validation (1-6 count)
- [ ] Add coordinator permission validation
- [ ] Add owner validation for update/delete
- [ ] Add logging for audit trail

### Testing
- [ ] Test create via JSON:API with service account
- [ ] Test author is correctly swapped to acting user
- [ ] Test tag validation (min/max, existence)
- [ ] Test coordinator permission enforcement
- [ ] Test update/delete ownership checks
- [ ] Test with invalid `X-Acting-User`

### Deployment
- [ ] Deploy module to staging
- [ ] Configure service account on staging
- [ ] Test with MCP server
- [ ] Deploy to production

---

## Testing Examples

### Create Announcement (cURL)

```bash
curl -X POST https://support.access-ci.org/jsonapi/node/access_news \
  -H "api-key: your-service-key" \
  -H "X-Acting-User: testuser@access-ci.org" \
  -H "Content-Type: application/vnd.api+json" \
  -H "Accept: application/vnd.api+json" \
  -d '{
    "data": {
      "type": "node--access_news",
      "attributes": {
        "title": "Test Announcement",
        "status": false,
        "body": {
          "value": "<p>Test content</p>",
          "format": "basic_html"
        }
      },
      "relationships": {
        "field_tags": {
          "data": [
            {"type": "taxonomy_term--tags", "id": "tag-uuid-here"}
          ]
        }
      }
    }
  }'
```

### Get Tags (for UUID lookup)

```bash
curl https://support.access-ci.org/jsonapi/taxonomy_term/tags \
  -H "Accept: application/vnd.api+json"
```

---

## Error Handling

The module should return appropriate JSON:API errors:

| Scenario | HTTP Status | Error |
|----------|-------------|-------|
| Missing `X-Acting-User` header | 400 | Custom error message |
| Acting user not found | 403 | Forbidden |
| Acting user not owner (update/delete) | 403 | Forbidden |
| Not coordinator for affinity group | 403 | Forbidden |
| Invalid tag count | 422 | Validation error |
| Non-existent tag | 422 | Validation error |

---

## Logging

All MCP operations should be logged:

```php
\Drupal::logger('access_mcp_author')->info(
  '@action on @type @id by @acting_user (via mcp_service)',
  [
    '@action' => 'create',
    '@type' => 'access_news',
    '@id' => $node->id(),
    '@acting_user' => $acting_user_id,
  ]
);
```

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| Service account compromise | Limit permissions to specific content types; rotate key regularly |
| Acting user spoofing | MCP server validates user JWT before setting header |
| Unauthorized content types | Hook only processes specific bundles (access_news) |
| Privilege escalation | Hook validates acting user permissions, not service account |

---

## Future: Events (Phase 2)

The same pattern applies to Events:
- Extend `access_mcp_author` to handle `eventseries` content type
- Add event-specific validation (dates, locations)
- Same service account, same authentication flow

---

## Related Documents

- [MCP Action Tools](./05-events-actions.md) - Overall action tools strategy
- [MCP Authentication](./06-mcp-authentication.md) - Authentication architecture
- [Key Auth Module](https://www.drupal.org/project/key_auth) - Drupal module documentation
