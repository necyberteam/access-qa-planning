# Capability Registry & User Experience

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [QA Bot Authentication](../archive/08-qa-bot-authentication.md) | [Researcher Profiles](./09-researcher-profiles.md) | [Analytics & Domain Agents](./10-analytics-and-domain-agents.md) | [Resource-Scoped Capabilities](./resource-scoped-capabilities.md)
>
> **Spec**: [Detailed implementation spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-18-capability-registry-design.md) in the access-agent repo

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

**`GET /api/v1/capabilities`** — fast, in-memory, no external calls. Returns all capabilities grouped by category. The few capabilities that still require authentication (acting on personal data or authored content) are visible to anonymous users but marked as locked with a login URL, so they know what's available after signing in.

**`GET /api/v1/capabilities/personalized`** — requires authentication, called lazily. Returns user-specific context like coordinator status and active allocations. Fed into the agent's system prompt, not surfaced directly in the UI.

Every capability is the **single unit of operator and runtime control**. Two environment variables (`ENABLED_CAPABILITIES` and `DISABLED_CAPABILITIES`) drive what the agent can do, and those controls propagate through the UI, the system prompt, the MCP tool catalog, RAG routing, domain agent routing, and usage attribution. See [Capabilities as Runtime Source of Truth](#capabilities-as-runtime-source-of-truth) below for the enforcement model.

### Categories

| Category | Label | Auth Required | Examples |
|----------|-------|---------------|----------|
| general | Ask a question | No | General Q&A about ACCESS |
| support | Get help | No | Open ticket, report login issue, report security |
| content | Manage content | Yes | Create/update/delete announcements |
| explore | Explore resources | No | Allocations, software, events, system status |
| analytics | Check usage | No* | XDMoD queries, usage metrics Q&A |

Most capabilities are available to anonymous users. The RAG pipeline accepts anonymous queries, gated by [Cloudflare Turnstile](./turnstile-bot-protection-spec.md) bot protection; authenticated users bypass the challenge. Only capabilities that act on a user's personal data or take authored actions (currently `manage_announcements`) require login.

*See [open questions](#open-questions) on whether `check_usage` should be authenticated to scope results to the user's allocations.

---

## Capabilities as Runtime Source of Truth

> **Status:** Proposed, April 2026. This section describes the target enforcement model. Implementation is tracked in the access-agent repo.

### Problem

Before this design, the capability registry was effectively a UI concern. The `DISABLED_CAPABILITIES` env var hid capabilities from discovery responses and the synthesis system prompt, but it did **not** prevent the planner from calling the underlying MCP tools, the rag_answer node from calling the RAG endpoint, or the router from invoking a domain agent. This meant:

- Operators had no single switch to turn a feature off
- The planner and the system prompt could drift (tools callable but not described, or described but unavailable)
- There was no way to run an "agent with no MCP tools" configuration for evaluation purposes without code changes
- Usage attribution and runtime behavior were maintained as two parallel hardcoded maps

### The model

Each capability declares **what it is backed by**. A capability's backend is one of:

- **`RagBackend(endpoint, scoped?)`** — the capability is served by a RAG lookup against the given endpoint (`general` or `xdmod`). `scoped=True` means the RAG call is resource-scoped to a specific RP.
- **`McpBackend(servers=[...])`** — the capability is served by calls to one or more MCP servers. All tools on those servers are considered part of this capability.
- **`None`** — the capability is pure prompt behavior with no external backend (reserved for future use; no current capabilities use this).

The registry aggregates backends across all enabled capabilities and exposes derived queries:

| Query | Returns |
|-------|---------|
| `enabled_mcp_servers()` | Union of `McpBackend.servers` across enabled capabilities |
| `enabled_rag_endpoints()` | Subset of `{"general", "xdmod"}` derived from enabled `RagBackend` capabilities |
| `scoped_rag_enabled()` | True if any enabled `RagBackend` has `scoped=True` |
| `is_domain_enabled(name)` | True if any capability belonging to that domain is enabled |

### Enforcement points

Three runtime components read from the registry. Each is the single gate for its layer.

| Component | Reads | Effect |
|-----------|-------|--------|
| **Tool catalog loader** | `enabled_mcp_servers()` | Tools from disabled servers are dropped from the catalog at load time. The planner never sees them. |
| **RAG answer node** | `enabled_rag_endpoints()`, `scoped_rag_enabled()` | RAG calls for disabled endpoints are skipped. Scoped RAG requires both the scoped flag and the base endpoint. |
| **Graph router + domain agent node** | `is_domain_enabled()` | If the classifier routes to a disabled domain, the router falls through to the general pipeline. The domain agent node also refuses as a defense-in-depth check. |

The UI (`/api/v1/capabilities`), the synthesis system prompt (`get_system_prompt_section`), and usage attribution (`infer_capability_id`) are all derived from the same registry, so they cannot drift from runtime behavior. When a capability is disabled:

- It disappears from the UI
- It disappears from the agent's self-description in the system prompt
- Its tools disappear from the planner's catalog
- Its RAG endpoint stops being called
- Its domain routing is bypassed
- It stops appearing in usage logs

### Operator interface

Two environment variables control the registry:

- **`ENABLED_CAPABILITIES`** (optional allow-list) — comma-separated capability IDs. If set, only listed IDs are candidates; everything else is filtered out. If unset, all capabilities defined in the registry are candidates.
- **`DISABLED_CAPABILITIES`** (always-wins deny-list) — comma-separated capability IDs. Applied after the allow-list. Disabled capabilities are removed no matter what.

**Semantics:** Deny always wins. If `ENABLED_CAPABILITIES=a,b,c` and `DISABLED_CAPABILITIES=c`, the active set is `{a, b}`. Overlap is legal; no warning is issued.

This gives operators:

| Scenario | Config |
|----------|--------|
| Normal production | Neither var set |
| Emergency disable one feature | `DISABLED_CAPABILITIES=manage_announcements` |
| Incremental rollout (early deployment) | `ENABLED_CAPABILITIES=ask_question,check_allocations,search_software` |
| Evaluation: RAG-only baseline | `ENABLED_CAPABILITIES=ask_question` |
| Evaluation: MCP-only (no RAG) | `DISABLED_CAPABILITIES=ask_question,ask_xdmod_question,ask_about_resource` (unusual; agent logs a warning) |

### Startup log

At registry build time the agent logs exactly what the filter resolved to, so operators can verify:

```
Capability filter: ENABLED=(all), DISABLED=(none)
Registry: 13 capabilities active
  Active: ask_question, ask_xdmod_question, ask_about_resource, check_allocations, ...
  → MCP servers in tool catalog: allocations, software-discovery, system-status, events, affinity-groups, xdmod, xdmod-data, nsf-awards, announcements, jsm
  → RAG endpoints enabled: general, xdmod
  → Scoped RAG enabled: yes
```

### RAG capabilities

Three new `RagBackend` capabilities are added to the general registry:

| Capability ID | Label (UI) | Backend | Category |
|---------------|------------|---------|----------|
| `ask_question` | Ask a question | `RagBackend(endpoint="general")` | general |
| `ask_xdmod_question` | Ask about usage metrics | `RagBackend(endpoint="xdmod")` | analytics |
| `ask_about_resource` | Ask about a specific resource | `RagBackend(endpoint="general", scoped=True)` | explore |

UI labels are deliberately generic — the underlying RAG provider is an implementation detail that users don't need to see.

### MCP capabilities

Existing general and domain capabilities each get an `McpBackend` declaration:

| Capability ID | Backend servers |
|---------------|-----------------|
| `check_allocations` | `allocations` |
| `search_software` | `software-discovery` |
| `check_system_status` | `system-status` |
| `browse_events` | `events` |
| `browse_affinity_groups` | `affinity-groups` |
| `check_usage` | `xdmod`, `xdmod-data` |
| `search_nsf_awards` | `nsf-awards` |
| `search_announcements`, `manage_announcements` | `announcements` |
| `open_ticket`, `report_login_problem`, `report_security` | `jsm` |

### Why per-server, not per-tool

Every capability today maps cleanly to one or more whole MCP servers. Per-tool granularity would force every capability to enumerate tool names, breaking every time a new tool is added to a server. The `McpBackend` dataclass is the right extension point if per-tool control becomes necessary later — add an optional `tools: list[str] | None` that defaults to "all tools from this server."

### Open questions

1. **`ask_question` naming.** Reuses the existing general-pipeline ID, now specifically meaning "RAG lookup against the general ACCESS docs endpoint." The name is short and doesn't leak implementation details, so retained.
2. **No-RAG warning.** When an operator disables every RAG capability, the agent logs a warning at startup (`"No RAG capabilities enabled — agent will rely on MCP tools only"`) but doesn't refuse to start. This is intentional — lets the eval pipeline run MCP-only comparisons and lets future scenarios disable RAG entirely if needed.
3. **Removing `DomainAgentConfig.mcp_servers` and `Capability.enabled`.** Both are now redundant with the backend-driven design. Kept for backward compat during rollout; removal is a follow-up cleanup.

### Evaluation use case

This design was motivated by the need to run production-readiness evals comparing the new agent against the current UKY-only prod system. With this design, the eval sequence becomes:

```bash
# Run 1: agent, RAG-only (agent without MCP value-add)
ENABLED_CAPABILITIES=ask_question \
  python -m src.eval run --questions eval/questions/friendly_battery.json

# Run 2: agent, full capabilities
python -m src.eval run --questions eval/questions/friendly_battery.json

# Compare
python -m src.eval compare --run-a <run1> --run-b <run2>
```

Both runs use the same agent binary, same classifier, same synthesis layer — only the enabled capability set differs. This isolates the value added by MCP tools on top of RAG. A third run against raw UKY RAG (via a separate runner mode in the eval pipeline) completes the three-way comparison; see the production baseline comparison section of the eval plan.

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

1. **RAG provider PII policy** — Can the user's name be included in the system prompt for authenticated users? Affects personalized greeting. Fallback: use ACCESS ID username or no name.
2. **`check_usage` authentication** — Currently `requires_auth=False` in code, but XDMoD queries arguably should be authenticated to scope results to the user's own allocations. Confirm intended behavior with the XDMoD team.
3. **RP slug mapping** — CiDeR resource IDs in Drupal may not match the RAG provider's valid RP slugs. Need a mapping between the two. See [Resource-Scoped Capabilities](./resource-scoped-capabilities.md) open question #1.

## Resolved Questions

| Question | Resolution |
|----------|------------|
| **Anonymous RAG access** | Live. Anonymous users can query the RAG pipeline, gated by [Cloudflare Turnstile](./turnstile-bot-protection-spec.md) bot protection. Authenticated users bypass the challenge. Most capabilities have been flipped to `requires_auth=False`. |
