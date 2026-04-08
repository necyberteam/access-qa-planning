# Resource-Scoped Capabilities

> **Related**: [Capability Registry](./11-capability-registry.md) | [Resource-Scoped RAG](./uky-resource-scoped-rag-spec.md) | [Agent Architecture](./01-agent-architecture.md)
>
> **Implements**: RP Documentation Plan Tasks 8-10 (`cyberteam_drupal/docs/superpowers/plans/2026-03-24-rp-documentation-pages.md`)

## Overview

The chatbot is embedded on resource provider (RP) documentation pages (e.g., Delta, Anvil). It acts as a full assistant that happens to know about that resource — same capabilities as the floating widget, but with RP context baked in. This spec covers how the capability registry and UKY resource-scoped RAG integrate to deliver that experience.

This is not a separate system. It extends the capability registry with a `resource_context` parameter so the same endpoint, same UI code, and same agent serve both the general floating widget and the RP-embedded chatbot.

## Design Decisions

1. **Two independent chatbot instances on RP pages.** The floating widget stays general. The embedded chatbot has RP context pre-loaded. Separate sessions, separate conversations. The embedded one advertises full capabilities (not just Q&A), so users discover the floating widget isn't the only way in.

2. **The agent owns section awareness.** The agent caches which documentation sections each RP has populated, fetched from the Drupal `/api/resources` endpoint. No per-request Drupal calls. The capabilities endpoint stays fast (<10ms).

3. **Unified capabilities endpoint.** `GET /api/v1/capabilities?resource_context=delta` returns capabilities filtered and augmented for that RP. Same endpoint the floating widget uses without the parameter. One endpoint, one UI flow — context shapes what you see.

4. **Adaptive layout.** The response includes a `layout` hint (`flat` or `categories`) based on total capability count. RP embeds with ≤8 items get a flat list of suggested questions. The floating widget with 15+ items gets category grouping. The UI renders based on the hint — no special "embedded mode" logic.

5. **RP context via data attribute.** The Drupal template puts `data-resource-context="delta"` on the embed div. `headerfooter.js` reads it and passes `resourceContext` to the `qaBot()` initialization. This keeps the chatbot portable — any page that knows the RP slug can embed it.

---

## How It Works

### 1. Embedding (Drupal)

The RP documentation template includes:

```html
<div class="embedded-qa-bot" data-resource-context="test-alpha-9999"></div>
```

`headerfooter.js` detects the `data-resource-context` attribute and passes it to the chatbot:

```js
qaBot({
  target: embeddedTarget,
  embedded: true,
  resourceContext: embeddedTarget.dataset.resourceContext,
  // ... other props
});
```

### 2. Capabilities Request (Chatbot → Agent)

On open, the chatbot calls:

```
GET /api/v1/capabilities?resource_context=delta
```

If no `resource_context`, returns the standard category-grouped capabilities (existing behavior).

### 3. Agent Builds RP-Scoped Response

The agent maintains an in-memory cache of RP section data:

```python
# Cache structure (refreshed on startup + TTL)
{
  "delta": {
    "title": "Delta",
    "populated_sections": ["login", "file_transfer", "storage", "queue_specs", "top_software", "datasets"]
  },
  "anvil": {
    "title": "Anvil",
    "populated_sections": ["login", "storage", "queue_specs"]
  }
}
```

**Cache source:** `GET /api/resources` on the Drupal site. The existing endpoint already returns `has_documentation` per resource. We extend it to include a `populated_sections` array listing which documentation sections have data.

**Cache refresh:** On agent startup, then every 30 minutes (configurable via `RP_CACHE_TTL_SECONDS`). The resource list is slow-changing data — documentation sections don't change hourly.

**Cache miss / invalid slug:** If `resource_context` isn't in the cache, ignore it and return the standard capabilities response. Don't error.

### 4. Section-to-Question Mapping

The agent maps populated sections to suggested questions:

| Section | Suggested Question |
|---------|-------------------|
| `login` | How do I log in? |
| `file_transfer` | How do I transfer files? |
| `storage` | What storage is available? |
| `queue_specs` | How do I submit a job? |
| `top_software` | What software is available? |
| `datasets` | What datasets are available? |

These are defined in the agent's capability registry code, not fetched from Drupal. Drupal says *which* sections exist; the agent decides *what to ask about them*.

**Always-present capabilities** (regardless of populated sections):
- "Get help with a [Resource Name] issue" (support/tickets)
- "Check my usage on [Resource Name]" (analytics, auth-required)

The resource name is interpolated from the cached title.

### 5. Capabilities Response

For `GET /api/v1/capabilities?resource_context=delta` with a fully-documented resource:

```json
{
  "resource_context": {
    "slug": "delta",
    "title": "Delta"
  },
  "layout": "flat",
  "capabilities": [
    {
      "id": "ask_about_resource",
      "label": "How do I log in?",
      "description": "Get help logging in to Delta",
      "section": "login"
    },
    {
      "id": "ask_about_resource",
      "label": "How do I transfer files?",
      "description": "Learn about file transfer options on Delta",
      "section": "file_transfer"
    },
    {
      "id": "ask_about_resource",
      "label": "What storage is available?",
      "description": "Explore storage options and quotas on Delta",
      "section": "storage"
    },
    {
      "id": "ask_about_resource",
      "label": "How do I submit a job?",
      "description": "Learn about queues and job submission on Delta",
      "section": "queue_specs"
    },
    {
      "id": "ask_about_resource",
      "label": "What software is available?",
      "description": "See frequently used software on Delta",
      "section": "top_software"
    },
    {
      "id": "ask_about_resource",
      "label": "What datasets are available?",
      "description": "Browse datasets available on Delta",
      "section": "datasets"
    },
    {
      "id": "open_ticket",
      "label": "Get help with a Delta issue",
      "description": "Create a support ticket",
      "requires_auth": false
    },
    {
      "id": "check_usage",
      "label": "Check my usage on Delta",
      "description": "View resource usage and performance data",
      "requires_auth": true,
      "locked": true
    }
  ],
  "is_authenticated": false,
  "login_url": "/login?redirect=..."
}
```

> **Note:** The `check_usage` capability is shown above as auth-required for illustration. In the current code it is `requires_auth=False`. See [open question on `check_usage` authentication](./11-capability-registry.md#open-questions). The RAG capabilities (`ask_about_resource`, `ask_question`) are not auth-required — anonymous users reach them through [Cloudflare Turnstile](./turnstile-bot-protection-spec.md) bot protection.

**Layout logic:** If `len(capabilities) <= 8`, set `layout: "flat"`. Otherwise, `layout: "categories"` and group into the standard category structure. Most RP embeds will have 5-8 items and get flat layout.

For the standard floating widget (no `resource_context`), the response is unchanged — categories with the full capability list per the existing spec.

### 6. Chatbot UI

The chatbot UI checks `response.layout`:

- `flat`: Render capabilities as a list of suggested-question buttons. Label them "Try asking:" to signal these are examples. Text input is always visible and prominent.
- `categories`: Render as category buttons (existing behavior from capability registry spec).

The `resource_context` from the page is included in every query the chatbot sends:

```json
POST /api/v1/query
{
  "query": "How do I submit a GPU job?",
  "resource_context": "delta"
}
```

### 7. Agent Query Processing

When `resource_context` is present in a query:

1. The classify node routes normally (RAG, domain agent, tools)
2. For RAG queries, `UKYClient.ask()` includes `rp_name` in the request body and sets `X-Origin` to the RP slug
3. If UKY responds out-of-scope (heuristic until `in_scope` field is available), retry without `rp_name` for general RAG
4. The fallback is transparent to the user — they get an answer either way
5. For non-RAG queries (tickets, XDMoD), the resource context is included as additional context in the agent's state but doesn't change routing

---

## Changes Required

### Drupal (`cyberteam_drupal`)

**`/api/resources` endpoint** — extend the list response to include `populated_sections` per resource:

```json
{
  "nid": 123,
  "title": "Delta",
  "resource_id": "delta",
  "populated_sections": ["login", "file_transfer", "storage", "queue_specs", "top_software"],
  "has_documentation": true
}
```

The sections are derived from checking which `field_rp_*` fields and paragraph references are non-empty.

**RP documentation template** — add `data-resource-context` to the embedded chatbot div (already partially implemented).

### Agent (`access-agent`)

- **RP section cache:** New `RPSectionCache` class that fetches and caches `/api/resources` data. Refreshed at startup + TTL.
- **Capabilities endpoint:** Accept optional `resource_context` query param. When present, build RP-scoped response using cache + section-to-question mapping.
- **Layout logic:** Count capabilities, return `layout: "flat"` or `"categories"`.
- **Query endpoint:** Accept optional `resource_context` in request body (per existing UKY scoped RAG spec).
- **UKY client:** Pass `rp_name` and `X-Origin` header (per existing UKY scoped RAG spec).

### Chatbot frontend (`qa-bot-core`, `access-qa-bot`)

- **`qa-bot-core`:** Accept `resourceContext` prop. Include in `resource_context` field on query POST. Pass to capabilities endpoint as query param.
- **`access-qa-bot`:** Accept and pass through `resourceContext` prop.
- **UI rendering:** Support `layout: "flat"` — render as suggested-question buttons with "Try asking:" label. Text input always visible.
- **`headerfooter.js`:** Read `data-resource-context` from embedded target, pass as `resourceContext` to `qaBot()`.

---

## UX Details

### Embedded Chatbot (RP Page)

1. User clicks "Ask about Delta" button → chatbot container expands
2. Welcome message: "I can help you with Delta. Here are some things to try, or ask me anything."
3. Suggested question buttons rendered flat (from capabilities endpoint)
4. Text input with placeholder: "Ask about Delta..."
5. User clicks a button or types — query includes `resource_context: "delta"`
6. Agent answers from RP-scoped RAG when possible, falls back to general

### Floating Widget (Same Page)

Unchanged. Standard category buttons, no resource context. Independent session.

### Sparse Resources

If an RP has minimal documentation (e.g., only a description), the capabilities response will have fewer suggested questions — just the always-present ones (support, usage). The flat layout still works. The chatbot is still useful for general ACCESS questions about that resource type.

---

## Open Questions

1. **RP slug mapping** — The `resource_id` in Drupal (e.g., `test-alpha-9999`) may not match UKY's valid RP slugs (e.g., `delta`, `anvil`). Need a mapping between CiDeR resource IDs and UKY slugs. This could live in the agent's config or be a field on the Drupal node.
2. **Welcome message personalization** — Should the embedded chatbot's welcome message reference the user's allocations on this specific resource (from the personalized capabilities endpoint)? e.g., "I see you have allocation TG-CIS123456 on Delta."
3. **UKY `in_scope` field** — Still pending from UKY. Until available, use text heuristic for fallback detection.
