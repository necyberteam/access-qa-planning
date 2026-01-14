# Review System

> **Part of the Data Pipeline**: [Q&A Data Preparation](./02-qa-data.md) → This doc
>
> **Related**: [Agent Architecture](./01-agent-architecture.md) | [Observability](./08-observability.md)

## Overview

Web application for reviewing Q&A data at two critical points:

1. **Pre-Deployment Review**: Approve Q&A pairs before they enter the RAG database
2. **Post-Deployment Feedback**: Capture user feedback on agent responses, flag bad answers for correction

Built on [Argilla](https://argilla.io), an open-source data annotation platform for LLM feedback and data curation, with a thin integration layer for ACCESS-specific features.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REVIEW SYSTEM SCOPE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PRE-DEPLOYMENT (batch extraction)       POST-DEPLOYMENT (live)             │
│  ─────────────────────────────────       ──────────────────────             │
│                                                                             │
│  ┌─────────────────────┐                 ┌─────────────────────┐            │
│  │ Python Extractors   │                 │ Agent (LangGraph)   │            │
│  │ - MCP extraction    │                 │ - User questions    │            │
│  │ - Doc ingestion     │                 │ - User feedback     │            │
│  │ - Embeddings/dedup  │                 │ - Thumbs up/down    │            │
│  └──────────┬──────────┘                 └──────────┬──────────┘            │
│             │                                       │                       │
│             │      Push directly to Argilla         │                       │
│             └──────────────────┬────────────────────┘                       │
│                                ▼                                            │
│                         ┌─────────────┐                                     │
│                         │   Argilla   │                                     │
│                         │   Review    │──▶ Approved Q&A → RAG Database      │
│                         └──────┬──────┘                                     │
│                                │                                            │
│                                ▼                                            │
│                   ┌────────────────────────┐                                │
│                   │ Auth Proxy + Notifier  │                                │
│                   │ - CILogon auth         │                                │
│                   │ - Slack/email alerts   │                                │
│                   └────────────────────────┘                                │
│                                │                                            │
│                                ▼                                            │
│                           Reviewers                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Argilla

| Factor | Benefit |
|--------|---------|
| **Production-ready UI** | Annotation interface with 80%+ of our requirements built-in |
| **Duplicate detection** | Built-in semantic search with vector embeddings |
| **Dataset versioning** | Track changes, export in multiple formats |
| **Self-hostable** | Docker deployment on existing infrastructure |
| **Active community** | Open-source with good documentation |

### What We Build

**Data flows directly to Argilla** from two sources:

| Source | Technology | Responsibility |
|--------|------------|----------------|
| **Python Extractors** | Python batch jobs | MCP extraction, doc ingestion, embeddings, dedup, push to Argilla, export approved data |
| **Agent** | LangGraph (Python) | Capture user Q&A and feedback during conversations, push to Argilla |

**Auth Proxy + Notifier** (thin service) handles Argilla integration:

| Responsibility | Description |
|----------------|-------------|
| **CILogon Authentication** | OAuth flow with ACCESS IdP, maps users to Argilla |
| **Notifications** | Slack/email alerts for pending reviews, monitors Argilla |
| **User Preferences** | Stores notification settings, domain assignments |
| **Real-Time Sync** | Webhook listener syncs approved Q&A to agent infrastructure (Redis + pgvector) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA SOURCES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────┐              ┌─────────────────────┐              │
│   │  Python Extractors  │              │  Agent (LangGraph)  │              │
│   │  (batch jobs)       │              │  (production)       │              │
│   │                     │              │                     │              │
│   │  - MCP extraction   │              │  - User questions   │              │
│   │  - Doc ingestion    │              │  - Feedback capture │              │
│   │  - Embeddings       │              │  - Thumbs up/down   │              │
│   │  - Dedup check      │              │                     │              │
│   │  - Export approved  │              │                     │              │
│   └──────────┬──────────┘              └──────────┬──────────┘              │
│              │                                    │                         │
│              │         Argilla Python SDK         │                         │
│              └─────────────────┬──────────────────┘                         │
│                                ▼                                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEPLOYMENT (Digital Ocean)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                    │
│   │   Argilla   │    │  Elastic-   │    │   Redis     │                    │
│   │   Server    │◀──▶│  search     │    │  (Argilla)  │                    │
│   │   :6900     │    │   :9200     │    │   :6379     │                    │
│   └──────┬──────┘    └─────────────┘    └─────────────┘                    │
│          │                                                                  │
│          │ Argilla UI + Webhooks                                            │
│          ▼                                                                  │
│   ┌──────────────────┐    ┌─────────────┐                                  │
│   │  Auth Proxy +    │◀──▶│ PostgreSQL  │  (user prefs, role mappings)     │
│   │  Notifier        │    │   :5432     │                                  │
│   │  :3010           │    └─────────────┘                                  │
│   │                  │                                                      │
│   │  - CILogon OAuth │───▶ Slack/Email                                     │
│   │  - Notifications │                                                      │
│   │  - Webhook sync  │───▶ Agent Redis (citations) + pgvector (RAG)        │
│   └────────┬─────────┘                                                      │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Nginx Proxy    │◀─── review.access-ci.org                              │
│   └─────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Datasets

### Pre-Deployment Review Dataset

For reviewing Q&A pairs generated from MCP extractions, documentation, and other sources before they enter the RAG database.

**Fields:**

| Field | Description |
|-------|-------------|
| question | The generated question |
| answer | The generated answer (markdown) |
| source_type | Origin: `mcp_extraction`, `user_qa`, `doc_generated` |
| domain | MCP server/category: `compute-resources`, `allocations`, `software`, etc. |
| source_data | Raw data preview the Q&A was derived from |
| citations | Citation markers (`<<SRC:...>>`) |

**Review Actions:**

| Action | Type | Description |
|--------|------|-------------|
| Review Decision | Label | approved, rejected, needs_edit, flagged |
| Rejection Reason | Label | duplicate, incorrect, vague, incomplete, citation_issue, other |
| Rejection Notes | Text | Free text explanation (if rejected) |
| Edited Question | Text | Modified question (if needs changes) |
| Edited Answer | Text | Modified answer (if needs changes) |
| Quality Rating | Rating | 1-5 scale |

**Filters:**
- Domain (routes to appropriate reviewers)
- Source type
- Complexity level

**Duplicate Detection:**
- Question embeddings with ~0.85 similarity threshold
- Flagged duplicates shown side-by-side for resolution

---

### Post-Deployment Feedback Dataset

For reviewing production responses that received negative user feedback.

**Fields:**

| Field | Description |
|-------|-------------|
| question | Original user question |
| response | Model's response that received feedback |
| conversation_context | Prior conversation turns (if multi-turn) |
| feedback_type | `thumbs_down`, `flagged`, `drift` |
| question_id | Links back to production logs / Grafana traces |
| session_id | Session for context |
| tools_used | List of MCP tools called to generate the response |

**Review Actions:**

| Action | Type | Description |
|--------|------|-------------|
| Review Action | Label | correct_answer, acceptable, add_to_database, dismiss |
| Corrected Answer | Text | The correct answer (if correcting) |
| Notes | Text | Reviewer notes |

**Filters:**
- Feedback type
- Tools used (routes to domain reviewers)

---

## User Roles & Workspaces

### Workspace Structure

Workspaces align with MCP servers/backend systems. Each team reviews Q&A related to their domain.

```
Argilla Workspaces
├── qa-review (Pre-deployment review)
│   ├── announcements (Support team)
│   ├── compute-resources (Operations team)
│   ├── allocations (Allocations team)
│   ├── events (Support team)
│   ├── software (Support team)
│   └── ... (one per MCP server)
│
└── feedback-review (Post-deployment)
    ├── announcements
    ├── compute-resources
    ├── allocations
    └── ... (mirrors qa-review structure)
```

### Reviewer Assignment

| Review Type | Assignment Logic |
|-------------|------------------|
| **Pre-deployment Q&A** | Based on `source_domain` - the MCP server the Q&A was extracted from |
| **Production feedback** | Based on `tools_used` - all teams whose tools were called can review |

For multi-tool queries (e.g., question that called both `compute-resources` and `allocations`), the feedback appears in both workspaces. Either team can review, or admins can coordinate.

### Role Mapping

| ACCESS Role | Argilla Role | Permissions |
|-------------|--------------|-------------|
| Admin | Owner | Full access, all workspaces, user management |
| Domain Reviewer | Annotator | Submit responses, view assigned domain workspace(s) |
| Viewer | Annotator (read-only) | View records, no submissions |

### CILogon Integration

1. User visits review site → redirected to CILogon
2. CILogon authenticates via ACCESS IdP → returns identity (`user@access-ci.org`)
3. Integration service maps ACCESS ID to Argilla user (creates if new)
4. Integration service assigns user to appropriate workspace(s) based on role config
5. User gets Argilla session, proxied through integration service

---

## Data Flows

### Ingestion: Batch Extraction → Argilla

Python extractors run as scheduled jobs or manually triggered:

1. Extractor fetches data from MCP servers or parses documents
2. LLM generates Q&A pairs with source_data for reviewer verification
3. Extractor generates question embeddings
4. Extractor checks for duplicates via Argilla SDK (`find_similar_records`)
5. If duplicate found, flag in metadata
6. Push record directly to Argilla qa-review dataset
7. Records appear in Argilla UI for review

### Ingestion: Production Feedback → Argilla

Agent captures feedback during live conversations:

1. User interacts with agent, asks questions
2. Agent logs Q&A pairs (potential additions to RAG database)
3. User gives thumbs-down or flags response
4. Agent pushes feedback record directly to Argilla feedback-review dataset
5. Record appears in Argilla UI for review
6. Positive feedback logged to Grafana only (not queued for human review)

### Sync: Argilla → QA Service

Approved Q&A pairs are synced to access-qa-service for RAG retrieval:

1. Sync triggered (manual, scheduled, or via webhook)
2. Query Argilla for approved records via SDK
3. For edited records, use edited version over original
4. Generate embeddings and upsert to pgvector database
5. New pairs immediately available for retrieval

### Real-Time Sync: Argilla → Agent Infrastructure

Approved Q&A pairs are synced in real-time to the agent's hallucination detection systems via Argilla webhooks.

**Why Real-Time Sync?**

The agent uses two systems that need approved Q&A data:

| System | Purpose | Sync Priority |
|--------|---------|---------------|
| **Citation Registry (Redis)** | Validates `<<SRC:domain:entity_id>>` citations to detect hallucinations | Critical - used on every static answer |
| **RAG Vector Store (pgvector)** | Semantic search fallback when hallucination detected | Secondary - only used on validation failure |

Both are updated together via the same webhook for simplicity - one code path, consistent state.

**Webhook Flow:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WEBHOOK-DRIVEN SYNC                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Argilla                      Auth Proxy + Notifier                         │
│  ───────                      ─────────────────────                         │
│                                                                             │
│  Reviewer approves ──webhook──► @webhook_listener("record.completed")       │
│  Q&A pair              │                                                    │
│                        ▼                                                    │
│              ┌─────────────────────────────────────────┐                    │
│              │  1. Parse approved Q&A record           │                    │
│              │  2. Extract domain + entity_id          │                    │
│              │  3. Add to Redis citation registry      │                    │
│              │  4. Generate question embedding         │                    │
│              │  5. Add to pgvector RAG store           │                    │
│              │  6. Increment approved_count            │                    │
│              └─────────────────────────────────────────┘                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Argilla Webhook Events:**

Argilla supports [webhook listeners](https://docs.argilla.io/latest/how_to_guides/webhooks/) for record events:

| Event | When Fired | Our Use |
|-------|------------|---------|
| `record.completed` | Record status → completed | Sync approved Q&A to agent |
| `record.updated` | Record modified | Re-sync if previously approved |
| `record.deleted` | Record removed | Remove from citation registry + RAG |

**Implementation in Auth Proxy + Notifier:**

```python
import argilla as rg
from argilla import webhook_listener

@webhook_listener(events=["record.completed"])
async def on_record_approved(record: rg.Record, type: str, timestamp: datetime):
    """Sync approved Q&A to agent infrastructure."""

    # Extract Q&A data
    question = record.fields["question"]
    answer = record.fields["answer"]
    domain = record.metadata["domain"]
    entity_id = extract_entity_id(record.metadata["source_ref"])

    # 1. Add to citation registry (Redis)
    await redis_client.set(f"citation:{domain}:{entity_id}", "1")

    # 2. Add to RAG vector store (pgvector)
    embedding = embedding_model.encode(question)
    await pg_pool.execute(
        "INSERT INTO qa_pairs (question, answer, domain, entity_id, question_embedding) ...",
        question, answer, domain, entity_id, embedding
    )

    # 3. Track approved count for monitoring
    await redis_client.incr("approved_count")
```

**Benefits:**
- **Immediate availability**: Approved Q&A usable for retrieval within seconds
- **No manual steps**: Reviewers just approve; sync happens automatically
- **Consistent state**: Citation registry and RAG store always match approved data

---

## Notification System

| Event | Recipients | Channel | Priority |
|-------|------------|---------|----------|
| New items pending review | Domain reviewers | Email + Slack | Normal |
| Review threshold reached (>N pending for >3 days) | Assigned reviewers | Slack @mention | Normal |
| Extraction failed | Admins | Email + Slack | High |
| Weekly summary | All users | Email | Low |

### User Preferences

Stored in integration service database:
- Email notification frequency (immediate / daily digest)
- Slack @mentions enabled
- Which domains they review

---

## Custom Features

Features beyond core Argilla functionality:

### Drift Detection (Python Extractors)

- Periodic job compares Q&A pairs against current MCP data
- If stored answer contradicts current data, record flagged for review
- Flagged when information may be outdated
- Frequency: Weekly or on-demand

### Implicit Feedback Tracking (Agent)

Signals logged for pattern analysis (not individual review):

| Signal | Interpretation |
|--------|----------------|
| Follow-up questions | May indicate incomplete answer |
| Session abandonment after response | Possible dissatisfaction |
| Copy/paste of response content | Positive signal |
| Time spent reading response | Engagement metric |
| Re-asking similar question | Possible confusion |

These are logged to Grafana for aggregate analysis. Only surfaced to Argilla review if systematic issues detected.

### Source Data Change Alerts (Python Extractors)

- Track hashes of source data from extractions
- When source changes, notify reviewers that derived Q&A pairs may need review
- Highlight changes since last extraction
- Link Q&A pairs to their source entities for traceability
- Option to regenerate Q&A for changed fields

### Review Actions for Flagged Responses

When a production response is flagged (thumbs down, drift, etc.):

| Action | Result |
|--------|--------|
| Correct the answer | Add corrected Q&A pair to RAG database |
| Mark as acceptable | Dismiss flag, no database change |
| Identify pattern | Create new Q&A pairs to address systematic gap |

---

## Implementation Phases

### Phase 1: Argilla Setup & Basic Review

- Deploy Argilla via Docker on existing droplet
- Configure Elasticsearch and Redis
- Create qa-review dataset with schema
- Set up admin users manually
- Test basic review workflow

**Deliverable:** Working Argilla instance with manual user management

---

### Phase 2: Python Extractors → Argilla

- Add Argilla SDK to Python extractors (`access-qa-extraction`)
- Implement embedding generation for duplicate detection
- Add `find_similar_records` dedup check before insert
- Push records directly to Argilla qa-review dataset
- Test end-to-end: extraction → Argilla → review

**Deliverable:** Extracted Q&A pairs appear in Argilla for review

---

### Phase 3: Auth Proxy + Notifier Service

- Create thin Node.js service for auth and notifications
- Add CILogon OAuth routes
- Implement CILogon → Argilla user mapping
- Set up Nginx proxy for auth flow
- Create workspace/role assignment logic
- Test login flow end-to-end

**Deliverable:** Reviewers can log in with ACCESS credentials

---

### Phase 4: Notifications

- Add Slack webhook integration to notifier service
- Implement pending count monitoring (poll Argilla)
- Create stale item alerts
- Add email notifications (optional)
- Set up weekly summary job
- Store user preferences in PostgreSQL

**Deliverable:** Reviewers notified of pending work

---

### Phase 5: Agent Feedback Integration

- Create feedback-review dataset in Argilla
- Add Argilla client to LangGraph agent
- Capture user Q&A pairs during conversations
- Push thumbs-down feedback to Argilla
- Implement drift detection in extractors (basic version)
- Test feedback → review → correction flow

**Deliverable:** User feedback flows into review queue

---

### Phase 6: Webhook Sync to QA Service

**Real-Time Sync (webhook-driven):**
- Add Argilla webhook listener to Auth Proxy + Notifier
- Listen for `record.completed`, `record.updated`, `record.deleted` events
- Sync approved Q&A to agent's Redis (citation registry) and pgvector (RAG store)

**Deliverable:**
- Approved Q&A synced to QA service in real-time
- New Q&A pairs immediately available for retrieval

---

## Resource Requirements

### Infrastructure (on existing droplet)

| Component | Memory | Storage | Notes |
|-----------|--------|---------|-------|
| Argilla Server | 1GB | 1GB | Python/FastAPI |
| Elasticsearch | 2GB | 10GB | Vector search, records |
| Redis | 256MB | 1GB | Caching |
| Auth Proxy + Notifier | 256MB | 100MB | Thin Node.js service |
| PostgreSQL | 512MB | 5GB | User prefs (may already exist) |

**Total additional:** ~4GB RAM, ~17GB storage

### External Services

- Slack webhook (free)
- Email service (existing Mailgun or similar)
- Embedding API (OpenAI or local model like all-MiniLM-L6-v2)

### Python Extractors (run separately)

- Can run locally, in CI, or as cron on any server with network access to Argilla
- Requires: Python 3.11+, Anthropic API key, Argilla API key
- No persistent infrastructure needed

---

## Open Questions

1. **Argilla version**: Use Argilla v1 (stable) or v2 (newer SDK)?
2. **Reviewer onboarding**: Who are the initial reviewers for each domain?

## Resolved Questions

| Question | Decision | Rationale |
|----------|----------|-----------|
| **Embedding model** | `sentence-transformers/all-MiniLM-L6-v2` | Fast, local, 384 dims, good quality for short Q&A |
| **Sync frequency** | Real-time webhook | Approved Q&A immediately available for retrieval |
| **Where does sync live** | Auth Proxy + Notifier service | Natural fit - it's the Argilla integration layer |

---

## References

- [Argilla Documentation](https://docs.argilla.io)
- [Argilla GitHub](https://github.com/argilla-io/argilla)
- [Argilla Docker Deployment](https://docs.argilla.io/latest/getting_started/quickstart/)
- [CILogon Documentation](https://www.cilogon.org/oidc)

---

> Approved Q&A pairs sync to [access-qa-service](https://github.com/necyberteam/access-qa-service) for RAG retrieval.
