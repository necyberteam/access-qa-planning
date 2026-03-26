# ACCESS Documentation Agent

> **Related**: [Q&A Data Preparation](./02-qa-data.md) | [Review System](./03-review-system.md) | [Model Training (Deprecated)](./04-model-training.md) | [Events Actions](./05-events-actions.md) | [Capability Registry](./11-capability-registry.md) | [Resource-Scoped RAG](./uky-resource-scoped-rag-spec.md)

## Overview

This is an AI-powered question-answering system for [ACCESS](https://access-ci.org) users. ACCESS (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support) allocates computing resources—supercomputers, cloud platforms, and storage systems—from Resource Providers to researchers across the US.

The agent uses a **RAG-primary architecture** with live MCP (Model Context Protocol) data access. Static knowledge comes from verified Q&A pairs retrieved via semantic search, while dynamic data comes from real-time API calls.

### Architecture Decision

After pilot testing, we pivoted from fine-tuning to RAG:
- Fine-tuned models didn't reliably retain factual details
- Worse, they hallucinated additional details around what they did memorize
- RAG retrieves verified answers directly - no hallucination risk
- RAG enables instant updates without retraining

### Current State (Production)

An intelligent agent system where:
- **Query Classifier**: LLM classifies queries as static, dynamic, combined, or domain
- **UKY Document RAG**: Primary RAG source — UKY's document retrieval endpoint is consulted for every query. Answers from curated documents with citations.
- **pgvector Q&A Pair RAG**: Secondary/future — human-curated Q&A pairs with semantic search. Currently disabled (stubs remain) pending production validation.
- **Live MCP Calls**: Handles dynamic queries (outages, events, metrics, user-specific data)
- **Combined Synthesis**: Merges UKY knowledge with MCP tool results for comprehensive answers
- **Direct Serve**: When tools add no value, UKY's answer is served directly without LLM rewrite
- **Parallel Execution**: Combined/dynamic queries run UKY RAG and tool planning concurrently
- **Domain Agents**: Specialized react agents for management operations (announcements, JSM tickets)
- **Citations Preserved**: Maintain link/source capability users rely on
- **Action Tools**: Authenticated operations via domain agents (announcements CRUD, ticket creation)

---

## Why RAG-Primary?

### The Problem with Fine-Tuning

Initial experiments with fine-tuned models showed:
- Models didn't reliably retain factual details from training data
- Worse, they hallucinated additional details around what they did memorize
- This mixing of real and fabricated information is worse than not knowing
- Retraining required for any knowledge update (slow iteration)

### The RAG Solution

RAG retrieval from verified Q&A pairs provides:
- Retrieves verified answers directly - no hallucination risk
- Instant updates (add Q&A pair → immediately available)
- Consistent citations (embedded in Q&A pairs)
- Clear "no match" signal when knowledge is missing

### Query Routing

| Query Type | Example | Handling |
|------------|---------|----------|
| **Static** | "What GPUs does Delta have?" | UKY document RAG → direct serve if confident |
| **Dynamic** | "Is Delta currently down?" | UKY RAG + MCP tools in parallel → synthesize |
| **Combined** | "What GPUs does Delta have and is it running?" | UKY RAG + MCP tools in parallel → combined synthesis |
| **Domain** | "Open a ticket about my login issue" | UKY RAG (context) → domain agent react loop |

Hardware/software/resource spec queries are classified as **combined** (not static) to preserve the MCP enrichment path — MCP tools often have more current data than documents.

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
│  LLM determines: STATIC | DYNAMIC | COMBINED | DOMAIN                       │
│  Also: rag_endpoint, domain (announcements/jsm), expanded_query             │
│  JSM only on explicit "open/file a ticket" language.                        │
│  Hardware/software specs → combined (not static).                           │
└──────┬──────────────────┬──────────────────────┬──────────┬─────────────────┘
       │                  │                      │          │
       ▼                  ▼                      ▼          ▼
┌────────────┐   ┌──────────────────┐   ┌────────────┐ ┌────────────┐
│   STATIC   │   │ COMBINED/DYNAMIC │   │   DOMAIN   │ │            │
│            │   │                  │   │            │ │  All paths │
│ Sequential │   │ Parallel path:   │   │ Sequential │ │  start     │
│ UKY RAG    │   │ UKY RAG + tool   │   │ UKY RAG    │ │  with UKY  │
│ first      │   │ planning run     │   │ → domain   │ │  document  │
│            │   │ concurrently     │   │   agent    │ │  RAG       │
└─────┬──────┘   └────────┬─────────┘   └─────┬──────┘ │            │
      │                   │                    │        └────────────┘
      ▼                   ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RESPONSE STRATEGIES                                  │
│                                                                             │
│  UKY DIRECT: confident static → serve raw UKY, hedge stripped, no rewrite  │
│  COMBINED:   UKY + MCP tools → LLM merges both sources                     │
│  TOOLS ONLY: MCP data only → LLM formats tool results                     │
│  DOMAIN:     react agent loop (JSM tickets, announcements CRUD)            │
│  FALLBACK:   domain agent fails → serve UKY answer from rag_matches        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RESPONSE                                        │
│                   (with citations, URLs preserved)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Details

```
┌─────────────────┐
│  access-agent   │  LangGraph orchestration (Python)
│  (qa.access-    │  Query classification → routing → synthesis
│   ci.org)       │  JWT cookie validation → user identity
└────────┬────────┘
         │ HTTP
         ├──────────────────────────────────────────────────────────┐
         ▼                                                          ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────┐
│  UKY Doc RAG   │     │  MCP Servers    │     │  pgvector QA Service    │
│  (primary RAG)  │     │  (TypeScript)   │     │  (FastAPI, port 8001)   │
│  access-ai-     │     │  10 servers     │     │  Currently disabled     │
│  grace1-ext.    │     │  incl. JSM      │     │  (stubs remain)         │
│  ccs.uky.edu    │     │                 │     └─────────────────────────┘
└─────────────────┘     └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │  ACCESS APIs    │
                        │  (live data)    │
                        └─────────────────┘
```

> **Authentication**: The agent validates user identity via a signed JWT cookie (`.access-ci.org` domain) for browser-based access, or via OAuth 2.1 for direct MCP clients (Claude, ChatGPT). Both paths result in a validated ACCESS ID passed to backends as `X-Acting-User`. See [QA Bot Authentication](./08-qa-bot-authentication.md) and [MCP Authentication](./06-mcp-authentication.md).

---

## Data Governance

What can be served via RAG vs. what must come from live MCP:

| Data Type | Source | Notes |
|-----------|--------|-------|
| Public resource specs (hardware, GPUs) | **RAG** | Core Q&A pairs |
| Public software modules | **RAG** | Versions change - consider freshness |
| Public NSF awards | **RAG** | Public record |
| General ACCESS documentation | **RAG** | From curated Q&A |
| Affinity group info | **RAG** | Stable |
| **Per-user allocations** | **MCP** | Privacy/accuracy - always live |
| **Project-specific details** | **MCP** | Privacy/accuracy - always live |
| **User SU balances** | **MCP** | Must be real-time |
| System outages/status | **MCP** | Real-time critical |
| Upcoming events | **MCP** | Changes frequently |
| Usage metrics (XDMoD) | **MCP** | Time-sensitive |

**Policy**: Never store user-specific, project-specific, or sensitive allocation details in the Q&A database.

---

## Query Classification

### LLM-Based Classification

The agent uses an LLM to classify queries based on intent:

**Static indicators** (→ RAG):
- Factual: "what is", "describe", "explain", "how does"
- Specs: "hardware", "specs", "specifications", "capabilities"
- Documentation: "how do I", "guide", "tutorial"

**Dynamic indicators** (→ UKY RAG + MCP tools in parallel):
- Time words: "currently", "right now", "today", "this week"
- Status words: "outage", "down", "maintenance", "status"
- User-specific: "my project", "my allocation", "my balance"
- Events: "upcoming", "next", "scheduled"

**Combined indicators** (→ UKY RAG + MCP tools in parallel):
- Multiple aspects: "specs AND status", "capabilities AND availability"
- Comparative with current state: "compare to what's available now"
- Hardware/software/resource specs (widened from static — MCP tools often have more current data)

**Domain indicators** (→ domain agent react loop):
- `domain=announcements`: User wants to CREATE/UPDATE/DELETE/MANAGE announcements
- `domain=jsm`: User explicitly asks to OPEN/FILE a ticket, REPORT an issue. Note: problem descriptions and complaints route to RAG, not JSM — only explicit ticket-filing language triggers domain routing

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

### UKY RAG Confidence

UKY handles its own retrieval thresholds internally. The agent evaluates UKY responses using **hedge/deflection detection** rather than similarity scores:

| Signal | Action |
|--------|--------|
| Confident answer (>500 chars or contains URLs/emails) | Serve directly, no LLM rewrite |
| Short answer with no links/substance | True deflection → fall through to MCP tools |
| Any response with domain=jsm/announcements | Save as context, route to domain agent |

This replaced the pgvector similarity threshold approach. UKY produces synthesized answers from documents, so the agent judges the response quality rather than raw similarity scores.

---

## RAG Retrieval

### UKY Document RAG (Primary)

The production RAG source is UKY's document retrieval endpoint at `access-ai-grace1-external.ccs.uky.edu`. UKY maintains its own document corpus and vector store. The agent calls it via HTTP with an API key (`ACCESS_AI_API_KEY`).

UKY returns synthesized answers from canonical ACCESS documents, including citations and links. The agent evaluates these answers for quality (hedge/deflection detection) and either serves them directly or combines them with MCP tool data.

### pgvector Q&A Service (Secondary — currently disabled)

FastAPI service providing semantic search over human-curated Q&A pairs:

| Feature | Implementation |
|---------|----------------|
| Vector store | PostgreSQL + pgvector |
| Index | HNSW (15x faster than IVFFlat) |
| Embeddings | sentence-transformers/all-MiniLM-L6-v2 |
| Caching | Query-level cache (90%+ hit rate for repeated queries) |

The pgvector path is disabled in production (stubs remain in `qa_client.py`). It was validated during the A.3 bake-off (see `SYSTEM_OVERVIEW.md`) — pgvector Q&A pairs outperform UKY 2-to-1 on entity-specific queries but have zero coverage on general how-to topics. Re-enabling is planned as E.4 (QAP production validation) once UKY+MCP is stable.

→ *Details: [Q&A Data Preparation](./02-qa-data.md)*

### Live MCP Servers

Handle dynamic data that must be current:

| Server | Data Type | Why Live |
|--------|-----------|----------|
| system-status | Outages, maintenance | Real-time critical |
| events | Upcoming workshops | Schedules change |
| announcements | News, updates | Time-sensitive |
| xdmod-metrics | Usage statistics | Always current |
| allocations (user-specific) | Balances, project status | Per-user, real-time |

→ *Details: [Events Actions](./05-events-actions.md)* for authenticated action patterns

---

## Response Synthesis

When queries involve both RAG and MCP data, outputs must be combined coherently.

### Synthesis Strategies

| Scenario | Strategy |
|----------|----------|
| **UKY confident, no tools needed** | Direct serve: raw UKY answer, hedge preamble stripped, no LLM rewrite |
| **UKY + MCP tools contributed** | LLM combined synthesis: merges UKY knowledge with MCP tool data |
| **Tools only (UKY deflected)** | LLM formats raw MCP tool data into readable answer |
| **Domain agent** | React agent loop handles response (announcements CRUD, ticket creation) |
| **Domain agent failed (no tools)** | Fallback to UKY answer from rag_matches |

The key principle: **avoid LLM rewrite when tools add no value**. If UKY provided a confident answer and MCP tools returned nothing useful (empty, failed, or error), the raw UKY answer is served directly. LLM synthesis only runs on the combined path where tools actually contributed data.

### Combined Response Pattern

```
User: "What GPU resources does Delta have and is it currently operational?"

1. RAG provides: GPU specs from verified Q&A (static)
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
│                           Q&A CURATION PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MCP Extraction ──┐                                                         │
│                   │                                                         │
│  User Feedback ───┼──▶ Argilla Review ──▶ Approved Q&A ──▶ QA Service      │
│                   │    (human review)                      (pgvector)       │
│  Documentation ───┘                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Query ──▶ Query Classifier ──▶ RAG Service ──────────┐                │
│                        │                                   │                │
│                        └──▶ Live MCP Servers ──────────────┼──▶ Response    │
│                                                            │                │
│                              Synthesize Node ◀─────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FEEDBACK LOOP                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Feedback (thumbs up/down) ──▶ Argilla ──▶ Q&A Updates                 │
│                                                                             │
│  Low-confidence answers ──▶ Review queue for human verification             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Answer correctness** | ≥90% vs ground-truth | Human eval on golden set |
| **RAG retrieval accuracy** | ≥85% precision | Relevance scoring |
| **Citation coverage** | ≥90% of answers include valid citation | Automated check |
| **Static query latency** | P95 < 500ms | Response time tracking |
| **Query routing accuracy** | ≥95% classification | Manual review sample |
| **User satisfaction** | ≥80% positive feedback | Thumbs up/down in UI |

---

## Observability & Logging

Every request through the agent logs structured data for monitoring and analysis.

### Request Logging

| Field | Purpose |
|-------|---------|
| `thread_id` | Conversation identifier |
| `query_text` | The user's question |
| `classification` | STATIC / DYNAMIC / COMBINED |
| `rag_matches` | Number and scores of RAG matches |
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
| Phase 3: RAG Service | ✅ Complete | access-qa-service with pgvector (currently disabled, stubs remain) |
| Phase 4: Agent Integration | ✅ Complete | LangGraph agent with UKY RAG + MCP |
| Phase 5: Production | ✅ Complete | Deployed and monitoring |
| Analytics Reporting | ✅ Complete | GA4 + PostgreSQL → weekly email reports via Mailgun |
| Domain Agent Routing | ✅ Committed (PR #1) | Announcements + JSM react agents on `uky-plus-mcp` branch |
| JWT Authentication | ✅ Complete | ES256 + JWKS cookie auth for browser-based access |
| Agent Graph Hardening | ✅ Committed (PR #1) | 14 commits: parallel RAG+plan, circuit breaker, hedge detection, direct serve, TOOL_CAVEATS |

### Current Focus

- **PR #1 (`uky-plus-mcp`)**: 14-commit branch validating UKY+MCP agent vs pure UKY. 18/18 battery pass. Ready for merge.
- **Capability registry**: 4 new specs published — capability discovery API, Turnstile bot protection, resource-scoped capabilities, resource-scoped RAG. Next major work after PR merge.
- **Analytics**: Weekly reports deployed and scheduled; remaining GA4 custom dimensions (`isEmbedded`, `chatbot_env`)

> *Details: [Analytics & Domain Agents](./10-analytics-and-domain-agents.md)*, [Capability Registry](./11-capability-registry.md)

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| RAG retrieval misses | Low-confidence answers flagged for review |
| Q&A pairs go stale | Regular sync from MCP servers + Argilla review |
| Query classifier mistakes | Default to COMBINED (safe fallback) |
| Citation accuracy | Citations embedded in Q&A pairs, validated on sync |
| Privacy concerns | Never store user-specific data in Q&A database |

### Fallback Hierarchy

When components fail, degrade gracefully:

```
1. RAG + MCP (optimal)
        │
        ▼ (RAG unavailable)
2. Live MCP only (no static knowledge, but accurate)
        │
        ▼ (MCP unavailable)
3. RAG only with disclaimer ("real-time data unavailable")
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
| Phase 1 | Announcements | **Active** - simpler content type to establish patterns |
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
| [QA Bot Authentication](./08-qa-bot-authentication.md) | JWT cookie auth for browser-based QA Bot |
| [Backend Integration Spec](./07-backend-integration-spec.md) | Service token + X-Acting-User contract |
| [Announcements API Spec](./drupal-announcements-api-spec.md) | Drupal developer spec for Phase 1 |
| [JSM MCP Server Plan](./jsm-mcp-server-plan.md) | Ticket creation/retrieval via agent |
| [JSM My Tickets API Spec](./jsm-my-tickets-api-spec.md) | Ticket lookup endpoint specification |
