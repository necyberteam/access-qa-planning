# Review System

> **Part of the Data Pipeline**: [Training Data](./02-training-data.md) → This doc → [Model Training](./04-model-training.md)
>
> **Related**: [Agent Architecture](./01-agent-architecture.md) | [Observability](./08-observability.md)

## Overview

Web application for reviewing Q&A data at two critical points:

1. **Pre-Training Review**: Approve Q&A pairs before they go into training data
2. **Post-Deployment Feedback**: Capture user feedback on model outputs, flag bad answers for correction

Built on [Argilla](https://argilla.io), an open-source data annotation platform for LLM feedback and training data curation, with a thin integration layer for ACCESS-specific features.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REVIEW SYSTEM SCOPE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PRE-TRAINING                              POST-DEPLOYMENT                  │
│  ────────────                              ───────────────                  │
│                                                                             │
│  MCP Extraction ──┐                        User asks question               │
│                   │                               │                         │
│  User Q&A DB ─────┼──▶ Argilla ──▶ Approved     │                         │
│                   │    Review     Training  ──▶ Model ──▶ Response         │
│  Documentation ───┘    UI         Data          │                         │
│                                                  ▼                         │
│                                           User feedback                     │
│                                           (thumbs up/down)                  │
│                                                  │                         │
│                                                  ▼                         │
│                                           Argilla ◀──── Flag bad answers   │
│                                           (correct & retrain)              │
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

### What We Build (Integration Layer)

Argilla doesn't provide everything we need. A thin Node.js integration service handles:

| Responsibility | Description |
|----------------|-------------|
| **CILogon Authentication** | OAuth flow with ACCESS IdP, maps users to Argilla |
| **Notifications** | Slack/email alerts for pending reviews |
| **Data Ingestion** | API to receive Q&A pairs from extraction pipeline |
| **Export** | Format approved records for training pipeline |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DEPLOYMENT                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Digital Ocean Droplet (existing)                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │   │
│   │   │   Argilla   │    │  Elastic-   │    │   Redis     │            │   │
│   │   │   Server    │◀──▶│  search     │    │  (vectors)  │            │   │
│   │   │   :6900     │    │   :9200     │    │   :6379     │            │   │
│   │   └──────┬──────┘    └─────────────┘    └─────────────┘            │   │
│   │          │                                                          │   │
│   │          │ API                                                      │   │
│   │          ▼                                                          │   │
│   │   ┌─────────────┐                                                   │   │
│   │   │ Integration │    ┌─────────────┐                                │   │
│   │   │   Service   │◀──▶│ PostgreSQL  │  (user mappings, preferences) │   │
│   │   │   :3010     │    │   :5432     │                                │   │
│   │   └──────┬──────┘    └─────────────┘                                │   │
│   │          │                                                          │   │
│   └──────────┼──────────────────────────────────────────────────────────┘   │
│              │                                                              │
│              ▼                                                              │
│   ┌─────────────────┐                                                       │
│   │  Nginx Proxy    │◀─── CILogon OAuth (via integration service)           │
│   │  review.access  │                                                       │
│   └─────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Datasets

### Pre-Training Review Dataset

For reviewing Q&A pairs generated from MCP extractions, documentation, and other sources before they enter training data.

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
| Review Action | Label | correct_answer, acceptable, add_to_training, dismiss |
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
├── qa-review (Pre-training review)
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
| **Pre-training Q&A** | Based on `source_domain` - the MCP server the Q&A was extracted from |
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

### Ingestion: Extraction → Argilla

1. Extraction pipeline generates Q&A pairs
2. Integration service receives pairs via API
3. For each pair:
   - Generate question embedding
   - Check for duplicates (vector similarity search)
   - If duplicate found, flag in metadata
   - Push record to qa-review dataset
4. Records appear in Argilla UI for review

### Ingestion: Production Feedback → Argilla

1. User gives thumbs-down in qa-bot-core
2. qa-bot-core calls integration service API
3. Integration service creates record in feedback-review dataset
4. Record appears in Argilla UI for review
5. Positive feedback logged to Grafana only (not queued for human review)

### Export: Argilla → Training Pipeline

1. Export triggered (manual or scheduled)
2. Integration service queries Argilla for approved records
3. For edited records, use edited version over original
4. Format as JSONL with metadata
5. Write to export location for training pipeline

---

## Notification System

| Event | Recipients | Channel | Priority |
|-------|------------|---------|----------|
| New items pending review | Domain reviewers | Email + Slack | Normal |
| Review threshold reached (>N pending for >3 days) | Assigned reviewers | Slack @mention | Normal |
| Extraction failed | Admins | Email + Slack | High |
| Retrain threshold reached | Admins | Email + Slack | High |
| Weekly summary | All users | Email | Low |

### User Preferences

Stored in integration service database:
- Email notification frequency (immediate / daily digest)
- Slack @mentions enabled
- Which domains they review

---

## Custom Features

These require implementation in the integration service (not provided by Argilla):

### Drift Detection

- Periodic job compares recent model responses against current MCP data
- If model answer contradicts current data, record added to feedback-review with `drift` type
- Flagged when model gives outdated information
- Triggers retrain consideration
- Frequency: Weekly or on-demand

### Implicit Feedback Tracking

Signals logged for pattern analysis (not individual review):

| Signal | Interpretation |
|--------|----------------|
| Follow-up questions | May indicate incomplete answer |
| Session abandonment after response | Possible dissatisfaction |
| Copy/paste of response content | Positive signal |
| Time spent reading response | Engagement metric |
| Re-asking similar question | Possible confusion |

These are logged to Grafana for aggregate analysis. Only surfaced to Argilla review if systematic issues detected.

### Source Data Change Alerts

- Track hashes of source data from extractions
- When source changes, notify reviewers that derived Q&A pairs may need review
- Highlight changes since last extraction
- Link Q&A pairs to their source entities for traceability
- Option to regenerate Q&A for changed fields

### Review Actions for Flagged Responses

When a production response is flagged (thumbs down, drift, etc.):

| Action | Result |
|--------|--------|
| Correct the answer | Add corrected Q&A pair to training data |
| Mark as acceptable | Dismiss flag, no training data change |
| Identify pattern | Create new training examples to address systematic issue |

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

### Phase 2: Data Ingestion Pipeline

- Create integration service scaffold
- Implement ingestion API endpoint
- Add embedding generation for duplicate detection
- Connect extraction pipeline to push to Argilla
- Test end-to-end: extraction → Argilla → review

**Deliverable:** Extracted Q&A pairs appear in Argilla for review

---

### Phase 3: CILogon Authentication

- Add OAuth routes to integration service
- Implement CILogon → Argilla user mapping
- Set up Nginx proxy for auth flow
- Create workspace/role assignment logic
- Test login flow end-to-end

**Deliverable:** Reviewers can log in with ACCESS credentials

---

### Phase 4: Notifications

- Implement Slack webhook integration
- Add pending count monitoring
- Create stale item alerts
- Add email notifications (optional)
- Set up weekly summary job

**Deliverable:** Reviewers notified of pending work

---

### Phase 5: Production Feedback Loop

- Create feedback-review dataset
- Add feedback ingestion endpoint
- Connect qa-bot-core thumbs down to Argilla
- Implement drift detection (basic version)
- Test feedback → review → correction flow

**Deliverable:** User feedback flows into review queue

---

### Phase 6: Export & Training Integration

- Implement export API
- Create training data format (JSONL)
- Add export scheduling
- Connect to training pipeline
- Document export process

**Deliverable:** Approved data exported for model training

---

## Resource Requirements

### Infrastructure (on existing droplet)

| Component | Memory | Storage | Notes |
|-----------|--------|---------|-------|
| Argilla Server | 1GB | 1GB | Python/FastAPI |
| Elasticsearch | 2GB | 10GB | Vector search, records |
| Redis | 256MB | 1GB | Caching |
| Integration Service | 512MB | 1GB | Node.js |
| PostgreSQL | 512MB | 5GB | Metadata (may already exist) |

**Total additional:** ~4GB RAM, ~18GB storage

### External Services

- Slack webhook (free)
- Email service (existing Mailgun or similar)
- Embedding API (OpenAI or local model like all-MiniLM-L6-v2)

---

## Open Questions

1. **Argilla version**: Use Argilla v1 (stable) or v2 (newer SDK)?
2. **Embedding model**: OpenAI embeddings vs local model?
3. **Reviewer onboarding**: Who are the initial reviewers for each domain?
4. **Export frequency**: On-demand vs scheduled exports?

---

## References

- [Argilla Documentation](https://docs.argilla.io)
- [Argilla GitHub](https://github.com/argilla-io/argilla)
- [Argilla Docker Deployment](https://docs.argilla.io/latest/getting_started/quickstart/)
- [CILogon Documentation](https://www.cilogon.org/oidc)

---

> *Next in pipeline: [Model Training](./04-model-training.md)* - What happens with approved data
