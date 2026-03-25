# Capability Registry & User Experience

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [QA Bot Authentication](./08-qa-bot-authentication.md) | [Researcher Profiles](./09-researcher-profiles.md) | [Analytics & Domain Agents](./10-analytics-and-domain-agents.md) | [Resource-Scoped Capabilities](./resource-scoped-capabilities.md)
>
> **Spec**: [Detailed implementation spec](../access-agent/docs/superpowers/specs/2026-03-18-capability-registry-design.md) in the access-agent repo

## Overview

As the QA system grows beyond general Q&A into domain-specific operations (announcements, tickets, XDMoD, allocations), users need a way to discover what's available, and the team needs structured data on how capabilities are being used. The capability registry addresses both.

Three components:
1. **Capability Registry** — A structured definition of everything the agent can do, served via API
2. **Personalized Context** — User-specific data (coordinator status, allocations) that makes the agent smarter
3. **Contextual Ratings** — Response ratings that route to the right backend and only appear when meaningful

---

## Capability Registry

### The Problem

The chatbot UI shows four hardcoded buttons ("Ask a question about ACCESS", "Open a Help Ticket", "Usage and performance of ACCESS resources (XDMoD)", "Report a security issue") that don't reflect the agent's actual capabilities. New features like announcements management are invisible. Internally, there's no structured way to track which capabilities are active or how they're performing.

### The Approach

Each capability gets a structured definition with a user-facing label, description, category, and auth requirement. These definitions live alongside the code that implements them — domain agent configs include their capabilities, general pipeline capabilities are defined in a separate registry.

The agent serves two endpoints:

**`GET /api/v1/capabilities`** — fast, in-memory, no external calls. Returns all capabilities grouped by category, filtered by auth status. Auth-required capabilities are visible to anonymous users but marked as locked (with a login URL), so they know what they're missing.

**`GET /api/v1/capabilities/personalized`** — requires authentication, called lazily. Returns user-specific context like coordinator status and active allocations. Fed into the agent's system prompt, not surfaced directly in the UI.

Capabilities can be disabled via environment variable (`DISABLED_CAPABILITIES`) without redeploying.

### Categories

| Category | Label | Auth Required | Examples |
|----------|-------|---------------|----------|
| general | Ask a question | Yes (UKY RAG) | General Q&A about ACCESS |
| support | Get help | No | Open ticket, report login issue, report security |
| content | Manage content | Yes | Create/update/delete announcements |
| explore | Explore resources | Yes (UKY RAG) | Allocations, software, events, system status |
| analytics | Check usage | Yes | XDMoD queries |

Most capabilities require authentication because the primary RAG pipeline (UKY) requires it. Only support capabilities (tickets, security reports) are available anonymously. If anonymous RAG access becomes available, the auth flags can be flipped without code changes.

---

## Chatbot UI Changes

### Current Flow
1. Welcome message, typing disabled
2. Four hardcoded buttons
3. User clicks button → transition message with AI disclaimer → typing enabled

### New Flow
1. Welcome message with AI disclaimer (combined, shown once), typing enabled from the start
2. Dynamic category buttons fetched from `/api/v1/capabilities`
3. "Show my options" discovery button (visually distinct, always last)
4. User either clicks a button or types freely — the agent handles both

**"Show my options"** sends as a message. The agent responds with a personalized capability summary — for authenticated users, it highlights relevant capabilities based on their context ("I see you coordinate Pegasus and have allocations on Delta..."). For anonymous users, it lists everything with login prompts for locked capabilities.

**Category buttons** send their label as a message. The agent responds conversationally with what it can help with in that category, rather than expanding to sub-buttons.

### Personalization Strategy

Personalized context feeds the agent's intelligence, not the UI. The agent uses it to give better responses:
- "Check my usage" → agent already knows the user's resources, offers specific options
- "Create an announcement" → agent knows they coordinate Pegasus, asks about group association
- "Show my options" → agent highlights capabilities relevant to this specific user

This scales naturally. A coordinator with 3 groups and 12 allocations doesn't overwhelm the button area — the detail lives in the conversation.

**Subtle discovery hints:**
- Input placeholder rotates through personalized suggestions ("Try: 'check my usage on Delta'")
- First visit gets a more descriptive welcome; return visits are shorter

### Branch Strategy

Changes happen on feature branches to avoid blocking other work:
- `access-agent`: `feature/capability-registry`
- `access-qa-bot`: `feature/dynamic-capabilities`
- `qa-bot-core`: `feature/capabilities-endpoint-prop` (if needed)

The `access-qa-bot` main branch stays available for the UKY RAG endpoint parameters feature.

---

## Personalized Context

### What's Available Today

| Data | Source | How It's Fetched |
|------|--------|-----------------|
| Coordinator status | Drupal `mcp_my_affinity_groups` view | jsonapi_views, filtered by `X-Acting-User` |
| Coordinated groups | Same view | Group names and IDs |
| Active allocations | Drupal `field_cider_resources` on user entity | JSON:API user endpoint |

### Future Extensions

| Data | Source | Status |
|------|--------|--------|
| Conversation summary | Agent usage logs | Planned |
| Researcher profile | Drupal user profile fields | Planned ([09-researcher-profiles](./09-researcher-profiles.md)) |
| Open tickets | JSM MCP server | Planned |

The personalized endpoint is designed to grow — each data source is fetched independently, failures are isolated, and partial responses are returned. New sources can be added without changing the endpoint contract.

### Caching

Results are cached in-memory per-user with a 5-minute TTL. Sufficient for single-instance deployment; can migrate to Redis if needed for scaling.

---

## Contextual Ratings

### The Problem

Rating buttons currently appear after every response and always route to UKY RAG. This is wrong for:
- Intermediate responses (clarifying questions, previews) — rating is meaningless
- Non-RAG responses (ticket creation, announcements) — UKY doesn't need this feedback
- Multi-turn workflows — rating mid-workflow is disruptive

### The Approach

The agent's response includes metadata:
- `is_final_response` — only show ratings when the agent has completed the user's request
- `rating_target` — route to UKY for RAG responses, to our own system for everything else
- `capability_id` — which capability was exercised, for analytics

Ratings include thumbs up/down plus optional free-text feedback. Stored in `usage_logs` linked by `query_id` (the existing `question_id`).

**When ratings appear:**
- After a RAG answer is delivered
- After a domain agent completes a task (announcement created, ticket opened)
- After tool results are summarized

**When ratings don't appear:**
- During clarifying questions
- When showing previews for confirmation
- During multi-turn workflows (gathering fields for announcement creation)

---

## Logging & Analytics

The capability registry enables capability-level analytics by adding to each log entry:
- `capability_id` — which capability was exercised
- `category` — which category it belongs to
- `was_authenticated` / `was_personalized` — context about the session
- `rating` / `rating_feedback` — user feedback when provided

This answers questions like:
- Which capabilities are most/least used?
- What's the success rate per capability?
- How do authenticated vs. anonymous usage patterns differ?
- When capabilities are added/removed, how does usage shift?
- Which capabilities get the worst ratings?

The capability ID is determined by the classify node (which already routes queries to domains) extended with a capability mapping.

---

## Security

- Capability endpoints never expose internal details (tool names, MCP servers, architecture)
- Auth-required capabilities are visible but not actionable for anonymous users — enforcement is server-side at the query endpoint
- Personalized endpoint requires authentication; data is fetched server-side, never from client input
- Existing `/api/v1/tools` and `/api/v1/catalog` endpoints must be restricted to admin access (they leak MCP server names)
- Disabled capabilities are invisible — not returned in any response

---

## Resource-Scoped Capabilities

The capabilities endpoint is extended to support resource provider (RP) context for the embedded chatbot on RP documentation pages. When called with `?resource_context=<slug>`, it returns RP-tailored suggested questions based on which documentation sections are populated, plus contextualized general capabilities (support, analytics).

See **[Resource-Scoped Capabilities](./resource-scoped-capabilities.md)** for the full integration design covering the capabilities endpoint extension, RP section caching, adaptive layout, and embedding approach.

---

## Open Questions

1. **UKY PII policy** — Can the user's name be included in the system prompt? Affects personalized greeting. Fallback: use ACCESS ID username or no name.
2. **Anonymous RAG access** — If UKY allows unauthenticated queries in the future, most capabilities can be unlocked for anonymous users by flipping the `requires_auth` flag.
3. **RP slug mapping** — CiDeR resource IDs in Drupal may not match UKY's valid RP slugs. Need a mapping between the two. See [Resource-Scoped Capabilities](./resource-scoped-capabilities.md) open question #1.
