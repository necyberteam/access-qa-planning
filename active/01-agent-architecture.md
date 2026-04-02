# ACCESS Documentation Agent

> **Related**: [Q&A Data Preparation](../archive/02-qa-data.md) | [Review System](./03-review-system.md) | [Model Training (Deprecated)](../archive/04-model-training.md) | [Events Actions](./05-events-actions.md) | [Capability Registry](./11-capability-registry.md) | [Resource-Scoped RAG](./uky-resource-scoped-rag-spec.md)

## Overview

This is an AI-powered question-answering system for [ACCESS](https://access-ci.org) users. ACCESS (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support) allocates computing resources—supercomputers, cloud platforms, and storage systems—from Resource Providers to researchers across the US.

The agent uses a **UKY-first architecture** with live MCP (Model Context Protocol) data access. All queries consult the UKY document RAG regardless of classification — the classification determines what additional data sources are consulted, not whether to skip RAG. Dynamic and combined queries run UKY RAG and MCP tool calls in parallel.

### Architecture Decisions

**Decision 001 — Fine-tuning to RAG**: After pilot testing, we pivoted from fine-tuning to RAG:
- Fine-tuned models didn't reliably retain factual details
- Worse, they hallucinated additional details around what they did memorize
- RAG retrieves verified answers directly - no hallucination risk
- RAG enables instant updates without retraining

**Decision 002 — UKY RAG over pgvector Q&A pairs**: The internal pgvector Q&A service (access-qa-service) is on hold. The University of Kentucky provides a hosted document RAG with broader coverage across ACCESS documentation. UKY RAG is the current production RAG backend.

**Decision 003 — UKY-first routing**: All queries consult UKY RAG. Classification determines parallelism: static queries use RAG only; dynamic and combined queries run RAG + MCP tool calls in parallel, then synthesize both results.

### Current State (Production)

An intelligent agent system where:
- **Query Classifier**: LLM classifies queries as static, dynamic, or combined; tags each query with a `capability_id` from the capability registry
- **UKY RAG**: All queries retrieve from UKY document RAG via semantic search
- **Live MCP Calls**: Dynamic and combined queries also invoke MCP tools in parallel with RAG (outages, events, metrics, user-specific data)
- **Combined Synthesis**: Merges UKY RAG results with tool results for comprehensive answers
- **Citations Preserved**: Maintain link/source capability users rely on
- **Domain Agents**: Authenticated operations via JSM (tickets) and Announcements domain agents — both implemented and deployed
- **Eval Pipeline**: LLM-as-judge scoring with Argilla for human review (Decision 005)
- **Turnstile**: Deferred bot-protection challenge for anonymous users
- **Feedback Loop**: User feedback flows to Argilla for curation

---

## Architecture Evolution

### Phase 1 — Fine-Tuning (Abandoned)

Initial experiments with fine-tuned models showed:
- Models didn't reliably retain factual details from training data
- Worse, they hallucinated additional details around what they did memorize
- This mixing of real and fabricated information is worse than not knowing
- Retraining required for any knowledge update (slow iteration)

### Phase 2 — pgvector Q&A RAG (On Hold)

RAG retrieval from an internal pgvector Q&A service (access-qa-service) addressed hallucination but had coverage limits — the Q&A pairs required ongoing manual curation to stay comprehensive.

### Phase 3 — UKY Document RAG (Current)

The University of Kentucky hosts a document RAG over ACCESS documentation with broader, more complete coverage. Migration to UKY RAG (Decision 002) replaced the pgvector Q&A pairs as the primary retrieval backend.

UKY RAG provides:
- Retrieves from full ACCESS documentation corpus — no gap from missing Q&A pairs
- Hosted and maintained by UKY, reducing internal ops burden
- Consistent citations from source documents
- Broader coverage than hand-curated Q&A pairs

### Query Routing

| Query Type | Example | Handling |
|------------|---------|----------|
| **Static** | "What GPUs does Delta have?" | UKY RAG only |
| **Dynamic** | "Is Delta currently down?" | UKY RAG + MCP tools (parallel) |
| **Combined** | "What GPUs does Delta have and is it running?" | UKY RAG + MCP tools (parallel) |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER QUERY                                      │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUERY CLASSIFIER                                   │
│                                                                             │
│  LLM determines: STATIC | DYNAMIC | COMBINED                                │
│  Tags query with capability_id from capability registry                     │
│  Based on: query intent, data requirements, user context                    │
└──────────┬──────────────────────┬──────────────────────┬────────────────────┘
           │                      │                      │
           ▼                      ▼                      ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│     STATIC       │   │    COMBINED      │   │     DYNAMIC      │
│                  │   │                  │   │                  │
│  UKY RAG only    │   │  UKY RAG +       │   │  UKY RAG +       │
│                  │   │  MCP (parallel)  │   │  MCP (parallel)  │
└────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SYNTHESIZE NODE                                    │
│                                                                             │
│  Combines UKY RAG results + tool results → coherent answer with citations   │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RESPONSE                                        │
│                         (with clickable citations)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Details

```
┌─────────────────┐
│  access-agent   │  LangGraph orchestration (Python)
│  (qa.access-    │  Query classification → routing → synthesis
│   ci.org)       │  JWT cookie validation → user identity
│                 │  Capability registry tagging (capability_id)
│                 │  Turnstile bot protection (anonymous users)
└────────┬────────┘
         │ HTTP
         ▼
┌─────────────────┐     ┌───────────────────────────┐     ┌─────────────────┐
│  UKY RAG        │     │  MCP Servers (TS+Python)  │     │  Domain Agents  │
│  (University    │     │  11 servers:              │     │  (Python/       │
│   of Kentucky)  │     │  compute-resources        │     │   LangGraph)    │
│  Document RAG   │     │  system-status            │     │  JSM (tickets)  │
│  over ACCESS    │     │  software-discovery       │     │  Announcements  │
│  documentation  │     │  xdmod                    │     │  (CRUD)         │
└─────────────────┘     │  allocations              │     └────────┬────────┘
                        │  xdmod-data (Python)      │              │
                        │  nsf-awards               │              ▼
                        │  events                   │     ┌─────────────────┐
                        │  announcements            │     │  Netlify Proxy  │
                        │  affinity-groups          │     │  → Atlassian    │
                        │  jsm                      │     │    JSM          │
                        │  xdmod-data               │     └─────────────────┘
                        └────────┬──────────────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │  ACCESS APIs    │
                        │  (live data)    │
                        └─────────────────┘

┌─────────────────┐     ┌─────────────────┐
│    Argilla      │     │  PostgreSQL     │
│  Human review   │     │  (usage logs,   │
│  LLM-as-judge   │     │  eval scores,   │
│  eval pipeline  │     │  checkpoints)   │
└─────────────────┘     └─────────────────┘
```

> **Authentication**: The agent validates user identity via a signed JWT cookie (`.access-ci.org` domain) for browser-based access, or via OAuth 2.1 for direct MCP clients (Claude, ChatGPT). Both paths result in a validated ACCESS ID passed to backends as `X-Acting-User`. See [QA Bot Authentication](../archive/08-qa-bot-authentication.md) and [MCP Authentication](./06-mcp-authentication.md).

---

## Data Governance

What can be served via RAG vs. what must come from live MCP:

| Data Type | Source | Notes |
|-----------|--------|-------|
| Public resource specs (hardware, GPUs) | **UKY RAG** | From ACCESS documentation corpus |
| Public software modules | **UKY RAG** | Versions change - MCP for current availability |
| Public NSF awards | **UKY RAG** | Public record |
| General ACCESS documentation | **UKY RAG** | Broad corpus coverage |
| Affinity group info | **UKY RAG** | Stable |
| **Per-user allocations** | **MCP** | Privacy/accuracy - always live |
| **Project-specific details** | **MCP** | Privacy/accuracy - always live |
| **User SU balances** | **MCP** | Must be real-time |
| System outages/status | **MCP** | Real-time critical |
| Upcoming events | **MCP** | Changes frequently |
| Usage metrics (XDMoD) | **MCP** | Time-sensitive |

**Policy**: Never store user-specific, project-specific, or sensitive allocation details in the RAG backend.

---

## Query Classification

### LLM-Based Classification

The agent uses an LLM to classify queries based on intent and tags each query with a `capability_id` from the capability registry. This enables per-capability analytics, eval scoring, and routing:

**Static indicators** (→ RAG):
- Factual: "what is", "describe", "explain", "how does"
- Specs: "hardware", "specs", "specifications", "capabilities"
- Documentation: "how do I", "guide", "tutorial"

**Dynamic indicators** (→ MCP tools):
- Time words: "currently", "right now", "today", "this week"
- Status words: "outage", "down", "maintenance", "status"
- User-specific: "my project", "my allocation", "my balance"
- Events: "upcoming", "next", "scheduled"

**Combined indicators** (→ RAG + MCP):
- Multiple aspects: "specs AND status", "capabilities AND availability"
- Comparative with current state: "compare to what's available now"

### Query Expansion

The classifier also **expands follow-up queries** into standalone questions by resolving pronouns and references from conversation history:

```
User: "What GPUs does Delta have?"
Agent: "Delta has NVIDIA A100 GPUs..."

User: "What about Expanse?"
      ↓ Expanded to:
      "What GPUs does Expanse have?"
```

This ensures RAG retrieval works correctly for follow-up questions that would otherwise lack context.

### Response Quality Checks

The agent evaluates whether UKY RAG responses are substantive or deflections. A "deflection" is a response that hedges without providing useful content (e.g., "I don't have information about that"). Substantive responses — even hedged ones — are kept if they contain URLs, email addresses, or significant content (>500 characters).

Note: The config still contains similarity thresholds (`RAG_THRESHOLD_STATIC`, etc.) from the pgvector era. These are not actively used with UKY RAG but are retained for potential future use if pgvector is re-enabled.

---

## RAG Retrieval

### UKY Document RAG

The University of Kentucky hosts a document RAG over ACCESS documentation. This is the current production RAG backend (Decision 002).

| Feature | Implementation |
|---------|----------------|
| Provider | University of Kentucky |
| Corpus | ACCESS documentation (broad coverage) |
| Query | All queries (static, dynamic, combined) |
| Routing | Always consulted; classification controls parallelism with MCP |

→ *Details: [Resource-Scoped RAG](./uky-resource-scoped-rag-spec.md)*

### pgvector Q&A Service (On Hold)

The internal `access-qa-service` (FastAPI + PostgreSQL + pgvector) remains available but is on hold while UKY RAG is the active backend. The Q&A pair format and HNSW index design are documented in the archive.

→ *Details: [Q&A Data Preparation](../archive/02-qa-data.md)*

### Live MCP Servers

11 MCP servers handle dynamic data that must be current. All use Streamable HTTP transport (migrated from SSE):

| Server | Data Type | Why Live |
|--------|-----------|----------|
| system-status | Outages, maintenance | Real-time critical |
| events | Upcoming workshops | Schedules change |
| announcements | News, updates | Time-sensitive |
| xdmod | Usage statistics | Always current |
| xdmod-data | Raw XDMoD data access | Always current |
| allocations | Balances, project status | Per-user, real-time |
| compute-resources | Hardware specs | Authoritative live source |
| software-discovery | Module availability | Changes with deployments |
| nsf-awards | Award data | Live NSF records |
| affinity-groups | Group membership/info | Live directory |
| jsm | Ticket operations | Live Atlassian JSM |

→ *Details: [Events Actions](./05-events-actions.md)* for authenticated action patterns

---

## Response Synthesis

When queries involve both RAG and MCP data, outputs must be combined coherently.

### Synthesis Strategies

| Query Type | Strategy |
|------------|----------|
| **Static** | Return UKY RAG answer directly (with citations) |
| **Dynamic** | Combine UKY RAG context + MCP tool results (both ran in parallel) |
| **Combined** | Combine UKY RAG context + MCP tool results (both ran in parallel) |
| **UKY direct** | When tools fail or aren't needed, serve UKY answer without LLM rewrite |

### Combined Response Pattern

```
User: "What GPU resources does Delta have and is it currently operational?"

1. UKY RAG provides: GPU specs from ACCESS documentation (static)
2. MCP provides: Current system status (dynamic)
3. Synthesizer combines: "Delta has NVIDIA A100 GPUs with 40GB memory
   (4 per node). The system is currently operational with no reported
   outages. [Source: compute-resources/delta.ncsa.access-ci.org]"
```

### Synthesis Guidelines

- Lead with the primary information source
- Integrate secondary source naturally
- Maintain single, coherent voice
- Preserve citations from RAG matches
- Include real-time status from tools

### Token Budget & Condensation

When tool results are very large (e.g., listing software across multiple resources), the synthesizer includes a **token budget check**:

| Setting | Default | Purpose |
|---------|---------|---------|
| `SYNTHESIS_TOKEN_BUDGET` | 80,000 | Max tokens for tool results before condensation |

If tool results exceed the budget:
1. An intermediate LLM call extracts only query-relevant information
2. The condensed results are passed to final synthesis
3. This prevents context overflow while preserving important details

```
Large tool results (261K tokens) → Condense → Relevant subset (8K tokens) → Synthesize
```

---

## Conversation History

Users often ask follow-up questions that depend on previous context:

```
User: "What GPUs does Delta have?"
Agent: "Delta has NVIDIA A100 GPUs..."

User: "How does that compare to Expanse?"  ← Requires knowing "that" = GPUs
Agent: "Compared to Delta's A100s, Expanse has..."
```

### Session Management

LangGraph maintains conversation state via PostgreSQL checkpointing:

| Field | Purpose |
|-------|---------|
| `thread_id` | Unique identifier for the conversation |
| `messages` | Full conversation history |
| `checkpoint` | State at each step for recovery |

---

## Scope & Rejection Handling

Not all queries are ACCESS-related. Handle out-of-scope queries gracefully.

### Helpful Rejection Pattern

Instead of a blunt "I can't help with that", provide guidance:

```
User: "What's the weather in Chicago?"

Agent: "I'm focused on ACCESS computing resources and can't help with
weather. But I can help you with:
- Finding compute resources for your research
- Checking system status and outages
- Understanding allocation policies

What ACCESS topic can I help you with?"
```

### Scope Categories

| Category | Action |
|----------|--------|
| ACCESS-related | Answer normally |
| Adjacent (HPC, research computing) | Answer if possible, clarify scope |
| Off-topic | Helpful rejection with examples |
| Harmful/inappropriate | Decline without examples |

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Query ──▶ Query Classifier ──▶ UKY RAG ──────────────┐                │
│                  (+ capability_id)         │                │                │
│                        │                  │ (parallel for   │                │
│                        └──▶ MCP Servers ──┘  dynamic/       │                │
│                                              combined)      │                │
│                                                             ▼                │
│                                              Synthesize Node ──▶ Response   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EVAL & FEEDBACK LOOP                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Feedback (thumbs up/down) ──▶ Argilla ──▶ Curation queue             │
│                                                                             │
│  LLM-as-judge eval (Decision 005) ──▶ Argilla ──▶ Human review             │
│                                                                             │
│  Low-confidence answers ──▶ Review queue for human verification             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Answer correctness** | ≥90% vs ground-truth | LLM-as-judge eval pipeline + human review in Argilla |
| **RAG retrieval accuracy** | ≥85% precision | Relevance scoring via eval pipeline |
| **Citation coverage** | ≥90% of answers include valid citation | Automated check |
| **Static query latency** | P95 < 500ms | Response time tracking |
| **Query routing accuracy** | ≥95% classification | Manual review sample |
| **User satisfaction** | ≥80% positive feedback | Thumbs up/down in UI |
| **Per-capability scores** | Tracked by capability_id | Eval pipeline aggregated by capability |

---

## Observability & Logging

Every request through the agent logs structured data for monitoring and analysis.

### Request Logging

| Field | Purpose |
|-------|---------|
| `thread_id` | Conversation identifier |
| `query_text` | The user's question |
| `classification` | STATIC / DYNAMIC / COMBINED |
| `capability_id` | Capability tag from capability registry |
| `rag_matches` | Number and scores of UKY RAG matches |
| `mcp_tools_called` | List of MCP tools invoked (if any) |
| `citations_returned` | Count of citations in response |
| `latency_ms` | End-to-end response time |
| `timestamp` | When the request was processed |

### Uses

- **Debugging**: Trace issues to specific requests
- **Performance monitoring**: Track latency, identify bottlenecks
- **Quality analysis**: Correlate RAG scores with user feedback
- **Q&A gap analysis**: Identify queries with no good RAG match

→ *Details: [Observability](./08-observability.md)*

---

## Implementation Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 0: Baseline | ✅ Complete | Golden eval set, current system analysis |
| Phase 1: Data Pipeline | ✅ Complete | MCP extraction, Argilla review system |
| Phase 2: Model Training | ⚠️ Deprecated | Fine-tuning abandoned for RAG |
| Phase 3: RAG Service | ⚠️ On Hold | access-qa-service with pgvector (superseded by UKY RAG) |
| Phase 4: Agent Integration | ✅ Complete | LangGraph agent with UKY RAG + MCP |
| Phase 5: Production | ✅ Complete | Deployed and monitoring |
| Analytics Reporting | ✅ Complete | GA4 + PostgreSQL → weekly email reports via Mailgun |
| Domain Agents | ✅ Complete | JSM (tickets) and Announcements (CRUD) deployed |
| JWT Authentication | ✅ Complete | ES256 + JWKS cookie auth for browser-based access |
| UKY RAG Migration | ✅ Complete | Migrated from pgvector Q&A pairs to UKY document RAG |
| Capability Registry | 🔄 In Progress | Data models and classification complete; API endpoints and UI integration in progress |
| Eval Pipeline | ✅ Complete | LLM-as-judge scoring with Argilla for human review |
| Streamable HTTP | ✅ Complete | MCP servers migrated from SSE to Streamable HTTP transport |
| Turnstile Bot Protection | 🔄 In Progress | Code complete; pending Cloudflare key provisioning for deployment |

### Current Focus

- **UKY RAG**: Active production backend; resource-scoped RAG spec in progress
- **Capability registry**: Queries tagged with `capability_id`; enabling per-capability eval and analytics
- **Eval pipeline**: LLM-as-judge scoring feeding Argilla for systematic quality review
- Improving synthesis quality for combined queries

> *Details: [Analytics & Domain Agents](./10-analytics-and-domain-agents.md)*

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| RAG retrieval misses | Low-confidence answers flagged for review; UKY corpus has broad coverage |
| UKY RAG corpus staleness | UKY maintains the document corpus; monitor for coverage gaps |
| Query classifier mistakes | Default to COMBINED (safe fallback) |
| Citation accuracy | Citations from UKY source documents; validated in eval pipeline |
| Privacy concerns | Never store user-specific data in RAG backend |
| Bot abuse | Turnstile deferred challenge for anonymous users |

### Fallback Hierarchy

When components fail, degrade gracefully:

```
1. UKY RAG + MCP (optimal)
        │
        ▼ (UKY RAG unavailable)
2. Live MCP only (no document knowledge, but accurate for dynamic queries)
        │
        ▼ (MCP unavailable)
3. UKY RAG only with disclaimer ("real-time data unavailable")
        │
        ▼ (both unavailable)
4. Base LLM with disclaimer ("I don't have current ACCESS data")
```

---

## Authenticated Actions

The architecture supports authenticated operations, enabling users to create content conversationally.

### Why CILogon

ACCESS already uses CILogon for authentication across its infrastructure. Using CILogon for agent authentication:
- Consistent with existing ACCESS identity management
- Supports 4000+ institutions via federated auth
- Users authenticate with their existing institutional credentials

### Authentication Pattern

| Component | Role |
|-----------|------|
| User | Authenticates via CILogon (institutional login) |
| MCP Server | Receives user identity, calls backend APIs |
| Backend | Service token auth + `X-Acting-User` header for attribution |

### Implementation Status

| Phase | Content Type | Status |
|-------|--------------|--------|
| Phase 1 | Announcements | **Deployed** - Announcements domain agent in production |
| Phase 1 | JSM Tickets | **Deployed** - JSM domain agent in production |
| Phase 2 | Events | Planned - more complex (dates, recurrence, locations) |

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Service token pattern | MCP servers use shared credentials for backend calls |
| User attribution | `X-Acting-User` header identifies who performed action |
| Draft-first | All API-created content starts unpublished |
| CILogon | Consistent with existing ACCESS authentication |

### Documentation

| Document | Purpose |
|----------|---------|
| [MCP Action Tools](./05-events-actions.md) | Overview of action tools initiative |
| [MCP Authentication](./06-mcp-authentication.md) | OAuth 2.1 for direct MCP clients (Claude, ChatGPT) |
| [QA Bot Authentication](../archive/08-qa-bot-authentication.md) | JWT cookie auth for browser-based QA Bot |
| [Backend Integration Spec](../archive/07-backend-integration-spec.md) | Service token + X-Acting-User contract |
| [Announcements API Spec](../archive/drupal-announcements-api-spec.md) | Drupal developer spec for Phase 1 |
| [JSM MCP Server Plan](../archive/jsm-mcp-server-plan.md) | Ticket creation/retrieval via agent |
| [JSM My Tickets API Spec](../archive/jsm-my-tickets-api-spec.md) | Ticket lookup endpoint specification |
