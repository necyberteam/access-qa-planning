# Drupal Announcements API Specification

> **For**: Drupal Developer
> **Module**: `access_mcp_author`
> **Priority**: Pilot for MCP Action Tools
> **Related**: [MCP Action Tools](./05-events-actions.md) | [MCP Authentication](./06-mcp-authentication.md) | [Backend Integration Spec](./07-backend-integration-spec.md)

## Overview

Enable the AI QA Bot and MCP clients to create, update, and delete announcements on behalf of authenticated users using **Drupal's built-in JSON:API** with a service account pattern.

---

## Current Implementation

The MCP announcements server (`@access-mcp/announcements`) already implements CRUD operations for announcements. This section documents the current state.

### How It Works Today

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         CURRENT IMPLEMENTATION                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐  │
│   │   MCP    │         │   MCP    │         │  Drupal  │         │  Drupal  │  │
│   │  Client  │         │  Server  │         │  Login   │         │ JSON:API │  │
│   └────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘  │
│        │                    │                    │                    │         │
│   1. Tool call              │                    │                    │         │
│        │───────────────────▶│                    │                    │         │
│        │                    │                    │                    │         │
│   2. Login with username/password (cookie session)                    │         │
│        │                    │───────────────────▶│                    │         │
│        │                    │◀──────────────────│                    │         │
│        │                    │   (session cookie + CSRF token)        │         │
│        │                    │                    │                    │         │
│   3. Look up user UUID by ACTING_USER_UID (env var)                   │         │
│        │                    │────────────────────────────────────────▶│         │
│        │                    │   GET /jsonapi/user/user?filter[drupal_internal__uid]=123
│        │                    │◀───────────────────────────────────────│         │
│        │                    │                    │                    │         │
│   4. Create announcement with uid relationship set                    │         │
│        │                    │────────────────────────────────────────▶│         │
│        │                    │   POST /jsonapi/node/access_news        │         │
│        │                    │   Cookie: SESS...                       │         │
│        │                    │   relationships: { uid: { id: user-uuid } }       │
│        │                    │◀───────────────────────────────────────│         │
│        │                    │                    │                    │         │
│   5. Response               │                    │                    │         │
│        │◀──────────────────│                    │                    │         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Current Configuration

| Setting | Value | Location |
|---------|-------|----------|
| `DRUPAL_API_URL` | `https://support.access-ci.org` | Environment variable |
| `DRUPAL_USERNAME` | Service account username | Environment variable |
| `DRUPAL_PASSWORD` | Service account password | Environment variable |
| `ACTING_USER_UID` | Drupal numeric user ID | Environment variable |

### Current Authentication

- **Method**: Cookie-based session authentication
- **Login**: POST to `/user/login?_format=json` with username/password
- **Per-request**: Session cookie + CSRF token in headers

### Current User Attribution

1. `ACTING_USER_UID` environment variable contains Drupal numeric UID (e.g., `12345`)
2. MCP server looks up user UUID via: `GET /jsonapi/user/user?filter[drupal_internal__uid]=12345`
3. MCP server sets the `uid` relationship in the JSON:API payload
4. Drupal stores the node with that user as owner

### Limitations of Current Approach

| Issue | Problem |
|-------|---------|
| Cookie auth | Sessions expire, require CSRF tokens, need re-auth handling |
| Numeric UID | Drupal-specific; other backends can't use this identifier |
| MCP does user lookup | Authorization logic scattered across MCP and Drupal |
| UID in env var | Static; doesn't support per-request user context |

---

## Target Implementation

Align with the [Backend Integration Spec](./07-backend-integration-spec.md) for a consistent pattern across all ACCESS backends.

### Target Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         TARGET IMPLEMENTATION                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐  │
│   │   MCP    │         │   MCP    │         │ Key Auth │         │  Drupal  │  │
│   │  Client  │         │  Server  │         │ (Drupal) │         │ JSON:API │  │
│   └────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘  │
│        │                    │                    │                    │         │
│   1. Tool call + Bearer token (user authenticated via CILogon)       │         │
│        │───────────────────▶│                    │                    │         │
│        │                    │                    │                    │         │
│   2. Validate MCP token, extract access_id                           │         │
│        │                    │                    │                    │         │
│   3. Call Drupal with service token + X-Acting-User header           │         │
│        │                    │───────────────────▶│                    │         │
│        │                    │   Authorization: Bearer {service_token} │         │
│        │                    │   X-Acting-User: jsmith@access-ci.org   │         │
│        │                    │   X-Request-ID: {uuid}                  │         │
│        │                    │                    │                    │         │
│   4. Key Auth validates service token            │                    │         │
│        │                    │                    │───────────────────▶│         │
│        │                    │                    │                    │         │
│   5. access_mcp_author hook:                     │                    │         │
│      - Parse X-Acting-User header                │                    │         │
│      - Look up user by username (= ACCESS ID)    │                    │         │
│      - Validate permissions                      │                    │         │
│      - Set node owner to acting user             │                    │         │
│        │                    │                    │                    │         │
│   6. Response               │                    │                    │         │
│        │◀───────────────────────────────────────────────────────────│         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Changes

| Aspect | Current | Target |
|--------|---------|--------|
| **Authentication** | Cookie session (username/password) | API key (Key Auth module) |
| **User identifier** | Drupal numeric UID | ACCESS ID (= Drupal username) |
| **User lookup** | MCP server does it | Drupal module does it |
| **Attribution** | MCP sets `uid` relationship | Drupal hook sets owner |
| **Tracing** | None | `X-Request-ID` header |

### User Identity Mapping

In Drupal, the **username field equals the ACCESS ID**:

```
Drupal username: jsmith@access-ci.org
ACCESS ID:       jsmith@access-ci.org
CILogon eppn:    jsmith@access-ci.org
```

This means the `access_mcp_author` module can look up users directly by username:

```php
$users = \Drupal::entityTypeManager()
  ->getStorage('user')
  ->loadByProperties(['name' => $access_id]);
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
| Roles | `mcp_service_role` (see permissions below) |

### 3. Configure Key Auth

1. Go to `/admin/config/system/key-auth`
2. Enable **Header** detection method
3. Set header name: `api-key` (or use `Authorization` with Bearer prefix)
4. Generate a key for the `mcp_service` user

### 4. Service Account Permissions

Create a role `mcp_service_role` with:

| Permission | Purpose |
|------------|---------|
| `access content` | View content |
| `create access_news content` | Create announcements |
| `edit any access_news content` | Edit announcements (author validated in hook) |
| `delete any access_news content` | Delete announcements (validated in hook) |
| `use jsonapi` | Access JSON:API endpoints |

**Note**: The actual authorization (can this user create/edit/delete?) is validated in the hook based on the acting user, not the service account.

---

## Module: `access_mcp_author`

A small module that:
1. Detects requests from the `mcp_service` account
2. Reads the `X-Acting-User` header (ACCESS ID)
3. Looks up Drupal user by username (which equals ACCESS ID)
4. Validates the acting user has permission for the action
5. Changes node ownership to the acting user

### Module Structure

```
web/modules/custom/access_mcp_author/
├── access_mcp_author.info.yml
├── access_mcp_author.module
├── access_mcp_author.services.yml
└── src/
    └── Service/
        └── ActingUserResolver.php
```

### Core Logic

**access_mcp_author.module**:

```php
<?php

use Drupal\node\NodeInterface;

/**
 * Implements hook_node_presave().
 */
function access_mcp_author_node_presave(NodeInterface $node) {
  // Only process for specific content types
  if (!in_array($node->bundle(), ['access_news', 'eventseries'])) {
    return;
  }

  // Only process requests from the MCP service account
  $current_user = \Drupal::currentUser();
  if ($current_user->getAccountName() !== 'mcp_service') {
    return;
  }

  // Get the acting user from header
  $request = \Drupal::request();
  $acting_user_id = $request->headers->get('X-Acting-User');

  if (empty($acting_user_id)) {
    throw new \Symfony\Component\HttpKernel\Exception\BadRequestHttpException(
      'X-Acting-User header is required for MCP service requests'
    );
  }

  // Look up user by username (username = ACCESS ID in Drupal)
  $users = \Drupal::entityTypeManager()
    ->getStorage('user')
    ->loadByProperties(['name' => $acting_user_id]);

  if (empty($users)) {
    throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException(
      'Acting user not found: ' . $acting_user_id
    );
  }

  $acting_user = reset($users);

  // For updates, verify ownership or admin status
  if (!$node->isNew()) {
    $is_owner = ($node->getOwnerId() === $acting_user->id());
    $is_admin = $acting_user->hasRole('administrator') ||
                $acting_user->hasRole('news_pm');

    if (!$is_owner && !$is_admin) {
      throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException(
        'Acting user does not have permission to edit this content'
      );
    }
  }

  // Validate affinity group coordinator permission (if applicable)
  if ($node->hasField('field_affinity_group_node') &&
      !$node->get('field_affinity_group_node')->isEmpty()) {
    _access_mcp_author_validate_coordinator($node, $acting_user);
  }

  // Set the owner to the acting user
  $node->setOwner($acting_user);

  // Log the operation
  \Drupal::logger('access_mcp_author')->info(
    '@action on @type by @acting_user (via mcp_service)',
    [
      '@action' => $node->isNew() ? 'create' : 'update',
      '@type' => $node->bundle(),
      '@acting_user' => $acting_user_id,
    ]
  );
}

/**
 * Implements hook_node_predelete().
 */
function access_mcp_author_node_predelete(NodeInterface $node) {
  // Similar validation for delete operations
  // Verify acting user owns the content or is admin
}

/**
 * Validate coordinator permission for affinity group.
 */
function _access_mcp_author_validate_coordinator(NodeInterface $node, $acting_user) {
  // Check if user is coordinator of the selected affinity group
  // Reuse logic from access_news_validate()
}
```

### Validation Logic

Reuse existing validation from `access_news.module`:

| Validation | Source | Apply To |
|------------|--------|----------|
| Tag count (1-6) | `access_news_tags_validate_element()` | Create, Update |
| Tags must exist | Field validation | Create, Update |
| Coordinator permission | `access_news_validate()` | Create, Update (when affinity group set) |
| Owner check | `access_mcp_author` hook | Update, Delete |

---

## JSON:API Endpoints

With JSON:API enabled for write operations (`read_only: false`), these endpoints become available:

### Create Announcement

**POST** `/jsonapi/node/access_news`

Headers:
```
api-key: <mcp_service_key>
X-Acting-User: jsmith@access-ci.org
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/vnd.api+json
```

Body:
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
- **No `uid` relationship needed** - the hook sets the owner based on `X-Acting-User`
- Relationships require UUIDs (MCP server fetches these via JSON:API reads)

### Update Announcement

**PATCH** `/jsonapi/node/access_news/{uuid}`

Same headers. Include only fields to update.

### Delete Announcement

**DELETE** `/jsonapi/node/access_news/{uuid}`

```
api-key: <mcp_service_key>
X-Acting-User: jsmith@access-ci.org
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

### Read Operations (for MCP Server)

These are already available (JSON:API is read-enabled):

| Endpoint | Purpose |
|----------|---------|
| `GET /jsonapi/taxonomy_term/tags` | List available tags (with UUIDs) |
| `GET /jsonapi/node/affinity_group` | List affinity groups |
| `GET /jsonapi/node/access_news?filter[uid.name]=jsmith@access-ci.org` | List user's announcements |

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

## Migration Path

### Phase 1: Add Key Auth (Parallel)

1. Install Key Auth module
2. Create `mcp_service` user with API key
3. Test that Key Auth works alongside existing cookie auth
4. MCP server can use either method during transition

### Phase 2: Add `access_mcp_author` Module

1. Create module with `X-Acting-User` header support
2. Module looks up user by username (ACCESS ID)
3. Module sets owner on presave
4. Test with manual curl requests

### Phase 3: Update MCP Server

1. Switch from cookie auth to API key auth
2. Switch from `ACTING_USER_UID` env var to `X-Acting-User` header
3. Remove user UUID lookup code (Drupal handles it now)
4. Remove `uid` relationship from JSON:API payload
5. Add `X-Request-ID` header generation

### Phase 4: Deprecate Old Pattern

1. Remove cookie auth code from MCP server
2. Remove `ACTING_USER_UID` configuration
3. Document new configuration requirements

---

## Configuration Changes

### Enable JSON:API Writes

Update `jsonapi.settings.yml`:

```yaml
read_only: false  # Changed from true
```

**Security consideration**: This enables writes for ALL content types via JSON:API. The `access_mcp_author` module validates that only authorized operations succeed.

---

## Implementation Checklist

### Drupal Setup
- [ ] Install and configure Key Auth module
- [ ] Create `mcp_service` user account
- [ ] Create `mcp_service_role` with appropriate permissions
- [ ] Generate API key for service account
- [ ] Enable JSON:API writes (or configure JSON:API Resources)

### Module Development
- [ ] Create `access_mcp_author` module
- [ ] Implement `hook_node_presave()` for author swap
- [ ] Implement `hook_node_predelete()` for delete validation
- [ ] Add `X-Acting-User` header parsing
- [ ] Add user lookup by username (ACCESS ID)
- [ ] Add tag validation (1-6 count)
- [ ] Add coordinator permission validation
- [ ] Add owner validation for update/delete
- [ ] Add `X-Request-ID` logging for audit trail

### Testing
- [ ] Test create via JSON:API with service account + API key
- [ ] Test author is correctly swapped to acting user
- [ ] Test tag validation (min/max, existence)
- [ ] Test coordinator permission enforcement
- [ ] Test update/delete ownership checks
- [ ] Test with invalid `X-Acting-User`
- [ ] Test with missing `X-Acting-User`

### MCP Server Updates
- [ ] Add API key authentication option
- [ ] Add `X-Acting-User` header (from validated MCP token)
- [ ] Add `X-Request-ID` header generation
- [ ] Remove `uid` relationship from payloads
- [ ] Remove user UUID lookup code
- [ ] Update configuration documentation

### Deployment
- [ ] Deploy module to staging
- [ ] Configure service account on staging
- [ ] Test with MCP server using new auth
- [ ] Deploy to production
- [ ] Switch MCP server to new auth method

---

## Testing Examples

### Create Announcement (Target Pattern)

```bash
curl -X POST https://support.access-ci.org/jsonapi/node/access_news \
  -H "api-key: your-service-key" \
  -H "X-Acting-User: jsmith@access-ci.org" \
  -H "X-Request-ID: $(uuidgen)" \
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

| Scenario | HTTP Status | Error Code |
|----------|-------------|------------|
| Missing `X-Acting-User` header | 400 | `BAD_REQUEST` |
| Acting user not found | 403 | `FORBIDDEN` |
| Acting user not owner (update/delete) | 403 | `FORBIDDEN` |
| Not coordinator for affinity group | 403 | `FORBIDDEN` |
| Invalid tag count | 422 | `VALIDATION_ERROR` |
| Non-existent tag | 422 | `VALIDATION_ERROR` |

Error response format:
```json
{
  "errors": [{
    "status": "403",
    "code": "FORBIDDEN",
    "title": "Access Denied",
    "detail": "Acting user does not have permission to edit this content",
    "meta": {
      "request_id": "550e8400-e29b-41d4-a716-446655440000"
    }
  }]
}
```

---

## Logging

All MCP operations should be logged with the request ID:

```php
\Drupal::logger('access_mcp_author')->info(
  '@action on @type @id by @acting_user (request: @request_id)',
  [
    '@action' => 'create',
    '@type' => 'access_news',
    '@id' => $node->id(),
    '@acting_user' => $acting_user_id,
    '@request_id' => $request->headers->get('X-Request-ID', 'unknown'),
  ]
);
```

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| Service account compromise | Limit permissions to specific content types; rotate key regularly |
| Acting user spoofing | Only accept header from authenticated `mcp_service` user |
| Unauthorized content types | Hook only processes specific bundles (access_news, eventseries) |
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
- [MCP Authentication](./06-mcp-authentication.md) - OAuth 2.1 authentication architecture
- [Backend Integration Spec](./07-backend-integration-spec.md) - Standard pattern for all backends
- [Key Auth Module](https://www.drupal.org/project/key_auth) - Drupal module documentation
