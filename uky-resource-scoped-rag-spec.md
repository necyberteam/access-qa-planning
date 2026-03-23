# Resource-Scoped RAG

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [Capability Registry](./11-capability-registry.md)

## Overview

When the chatbot is embedded on a resource-specific page (e.g., Anvil documentation), queries are routed to a resource-specific vector database for more focused answers. If the question falls outside that resource's scope, the agent falls back to the general RAG pipeline.

## UKY RAG API

The UKY endpoint already supports RP-scoped queries:

- **`rp_name` in request body** — routes to that RP's vector database. Omit for general ACCESS queries.
- **`X-Origin` header** — set to the RP slug for scoped queries, or `access` for general. Used for analytics.
- Invalid slugs return 400 with `{"error": "Unknown resource provider: xyz"}`.

Valid RP slugs: aces, ama27, anvil, aws, bridges2, cloudbank, delta, deltaai, derecho, expanse, fabric, googlecloud, granite, ibmcloud, jetstream2, kyric, launch, microsoftazure, neocortex, nexus, osn, osg, pnrp, ranch, repacss, sage, sgx3, stampede3, voyager.

Example scoped request:
```json
POST /access/chat/api/
Headers: X-API-KEY, X-Session-ID, X-Origin: delta
Body: {"query": "How do I submit a GPU job?", "rp_name": "delta"}
```

### Outstanding: `in_scope` response field

The API does not yet return an `in_scope` flag. When a question is outside the RP's scope, the response contains a text message saying so. We've requested a boolean `in_scope` field in the response to enable reliable fallback detection without parsing message text.

## Changes Required

### Agent (access-agent)

- Add `rp_name` parameter to `UKYClient.ask()` — included in the request body when present
- Set `X-Origin` header to the RP slug (currently hardcoded to `access-agent`)
- Add `in_scope` field to `UKYResponse` model (optional, defaults to `true` until UKY implements it)
- When `in_scope` is `false`, retry with `rp_name` omitted (general collection)
- Accept `resource_context` in the query API and pass it through

### Chatbot frontend

- `access-qa-bot`: Accept `resourceContext` prop, include in query payload
- `qa-bot-core`: Pass `resource_context` field in the POST body to `/api/v1/query`
- Drupal embedding (`headerfooter.js`): Set `resourceContext` based on the page (e.g., resource documentation pages pass the RP slug)

### Fallback behavior

Until UKY adds the `in_scope` field, we can use a heuristic: if the scoped response contains phrases like "outside the scope" or "I don't have information about that", retry with general RAG. Once the `in_scope` field is available, switch to using that.

### UX

- Resource-specific pages label the chatbot contextually (e.g., "Ask about Anvil")
- Out-of-scope questions fall back to general RAG — the user gets an answer either way
- The fallback is transparent to the user
