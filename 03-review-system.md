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

| Field | Type | Description |
|-------|------|-------------|
| question | text | Original user question |
| response | text | Model's response that received feedback |
| conversation_context | text[] | Prior conversation turns (if multi-turn) |
| feedback_type | enum | `thumbs_down`, `thumbs_up` |
| feedback_tags | text[] | User-selected reasons: `factually_wrong`, `outdated`, `incomplete`, `other` |
| user_comment | text | Optional free-text comment from user |
| question_id | uuid | Links back to production logs / Grafana traces |
| session_id | uuid | Session for context |
| tools_used | text[] | List of MCP tools called to generate the response |
| rag_matches | json | RAG retrieval results (question, score) for debugging |
| trace_id | string | OpenTelemetry trace ID for Grafana correlation |
| timestamp | datetime | When feedback was submitted |

**Review Actions:**

| Action | Type | Description |
|--------|------|-------------|
| Review Action | Label | `correct_answer`, `acceptable`, `add_to_database`, `dismiss` |
| Corrected Answer | Text | The correct answer (if correcting) |
| Priority | Label | `high`, `normal`, `low` (auto-set based on feedback_tags) |
| Notes | Text | Reviewer notes |

**Auto-Priority Rules:**

| Feedback Tag | Auto-Priority | Rationale |
|--------------|---------------|-----------|
| `factually_wrong` | High | Incorrect info is urgent |
| `outdated` | Normal | May need source refresh |
| `incomplete` | Normal | RAG/synthesis tuning |
| `other` | Normal | Needs triage |
| (no tag, just thumbs_down) | Low | Less actionable without context |

**Filters:**
- Feedback type
- Feedback tags (new)
- Priority (new)
- Tools used (routes to domain reviewers)
- Date range

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

> **Note: Access Control Granularity**
>
> Argilla supports access control at the workspace and dataset level, but not at the individual record level. Currently, reviewers filter by `tools_used` metadata to see their domain's feedback, but they *can* see other teams' records if they remove the filter.
>
> If strict team isolation becomes a requirement, options include:
> - **Separate datasets per team** (e.g., `feedback-compute`, `feedback-allocations`) with team members assigned only to their dataset
> - **Separate workspaces per team** for full isolation
>
> The current single-dataset approach is simpler to maintain. Splitting is straightforward if needed later.

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

---

## Feedback Collection

### Feedback Types

Based on [research into conversational AI feedback](https://denser.ai/blog/chatbot-feedback/) and [implicit feedback in LLM dialogues](https://arxiv.org/html/2507.23158), we collect multiple feedback signals:

| Type | Signal | Collection | Destination |
|------|--------|------------|-------------|
| **Explicit** | Thumbs up/down | User action in UI | Argilla (thumbs-down) / Grafana (thumbs-up) |
| **Explicit** | Written comment | Optional text field | Argilla with feedback record |
| **Explicit** | Flag as harmful/wrong | User action in UI | Argilla (priority review) |
| **Behavioral** | Follow-up clarification | Agent detects rephrasing | Grafana (aggregate analysis) |
| **Behavioral** | Session abandonment | No response after agent reply | Grafana (aggregate analysis) |
| **Behavioral** | Query repetition | Same/similar question re-asked | Grafana (aggregate analysis) |
| **Behavioral** | Escalation request | User asks for human help | Grafana + potential Argilla flag |

### Explicit Feedback: What Users Can Do

| Action | UI Element | Result |
|--------|------------|--------|
| **Thumbs up** | 👍 button on response | Logged to Grafana; no review queue |
| **Thumbs down** | 👎 button on response | Pushed to Argilla for review |
| **Add comment** | Text field (appears after thumbs down) | Attached to Argilla record |
| **Flag as wrong** | "This is incorrect" link | Pushed to Argilla, tagged `factually_wrong` |
| **Flag as harmful** | "This is inappropriate" link | Pushed to Argilla, tagged `harmful`, priority queue |

### Implicit Feedback: Behavioral Signals

These are logged automatically—no user action required. Based on [Meta's RLUF research](https://arxiv.org/html/2505.14946), behavioral signals can indicate satisfaction without interrupting the user.

| Signal | How Detected | Interpretation | Caution |
|--------|--------------|----------------|---------|
| **Rephrasing** | User's next message is semantically similar to previous | Possible confusion or incomplete answer | Could be refinement, not dissatisfaction |
| **Abandonment** | Session ends within 30s of response, no further interaction | Possible dissatisfaction | User may have gotten what they needed |
| **Copy action** | User copies response text (if detectable) | Positive—found useful | Browser-dependent |
| **Escalation** | User says "talk to human" / "contact support" | Agent couldn't help | Clear negative signal |
| **Quick follow-up** | Next question within 10s | Engaged, possibly incomplete answer | Could be multi-part query |

**Important**: [Research shows](https://arxiv.org/html/2507.23158) implicit feedback is "noisy as a learning signal." We use it for **aggregate pattern detection**, not individual response evaluation.

### Feedback API

The agent exposes a feedback endpoint for the chat UI:

**Endpoint**: `POST /api/feedback`

**Request**:
```json
{
  "session_id": "uuid",
  "message_id": "uuid",
  "feedback_type": "thumbs_down",
  "comment": "The GPU count was wrong",
  "tags": ["factually_wrong"]
}
```

**Feedback Types**:
| Type | Description |
|------|-------------|
| `thumbs_up` | Positive rating |
| `thumbs_down` | Negative rating |
| `flag_wrong` | Factually incorrect |
| `flag_harmful` | Inappropriate content |
| `flag_outdated` | Information is stale |

**Response**:
```json
{
  "status": "received",
  "feedback_id": "uuid"
}
```

**Processing Flow**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FEEDBACK PROCESSING FLOW                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Chat UI                     Agent                      Destinations       │
│   ───────                     ─────                      ────────────       │
│                                                                             │
│   User clicks 👎 ───────────► POST /api/feedback                            │
│   + optional comment                  │                                     │
│                                       ▼                                     │
│                               ┌───────────────┐                             │
│                               │ Enrich with:  │                             │
│                               │ - Full Q&A    │                             │
│                               │ - Tools used  │                             │
│                               │ - RAG scores  │                             │
│                               │ - Trace ID    │                             │
│                               └───────┬───────┘                             │
│                                       │                                     │
│                         ┌─────────────┼─────────────┐                       │
│                         ▼             ▼             ▼                       │
│                   ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│                   │ Argilla  │  │ Grafana  │  │ Drupal   │                  │
│                   │ (review) │  │ (metrics)│  │ (future) │                  │
│                   └──────────┘  └──────────┘  └──────────┘                  │
│                                                                             │
│   thumbs_down, flag_* ────────► Argilla feedback-review dataset             │
│   All feedback ───────────────► Grafana (metrics, dashboards)               │
│   If AI profile enabled ──────► Drupal /api/ai-profile (future)             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Argilla Record Structure

When feedback is pushed to Argilla:

```json
{
  "question": "What GPUs does Delta have?",
  "response": "Delta has NVIDIA V100 GPUs...",
  "conversation_context": ["prior", "messages"],
  "feedback_type": "thumbs_down",
  "user_comment": "It's A100s, not V100s",
  "tags": ["factually_wrong"],
  "tools_used": ["compute-resources"],
  "rag_matches": [{"question": "...", "score": 0.87}],
  "trace_id": "abc123",
  "session_id": "xyz789",
  "timestamp": "2025-01-30T10:00:00Z"
}
```

### Grafana Metrics

All feedback (positive and negative) logged to Grafana for dashboards:

| Metric | Description |
|--------|-------------|
| `feedback_total` | Count by type (thumbs_up, thumbs_down, flag_*) |
| `feedback_by_tool` | Breakdown by MCP tools used |
| `feedback_by_domain` | Breakdown by Q&A domain |
| `satisfaction_rate` | thumbs_up / (thumbs_up + thumbs_down) |
| `escalation_rate` | Requests for human help |
| `abandonment_rate` | Sessions ending without resolution |

### Chat UI Implementation

**Repository**: `qa-bot-core` (React component library)

**Current State**: Simple thumbs up/down with `rating: 1` or `rating: 0`

**Target State**: Expanded feedback with reason selection and optional comment

#### UI Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FEEDBACK UI FLOW                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  After bot response, show:                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Was this helpful?   [👍 Yes]   [👎 No]                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  If 👍 clicked:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Thanks for the feedback! Feel free to ask another question.        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│  → Send: { feedback_type: "thumbs_up" }                                    │
│                                                                             │
│  If 👎 clicked, expand to show:                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  What was the issue?                                                │   │
│  │                                                                     │   │
│  │  ○ This is incorrect                                                │   │
│  │  ○ This is outdated                                                 │   │
│  │  ○ This didn't answer my question                                   │   │
│  │  ○ Other                                                            │   │
│  │                                                                     │   │
│  │  [Optional: Add details ________________________]                   │   │
│  │                                                                     │   │
│  │  [Submit feedback]   [Skip]                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  If user selects reason + submits:                                         │
│  → Send: { feedback_type: "thumbs_down", tags: ["factually_wrong"],        │
│            comment: "user's optional comment" }                            │
│                                                                             │
│  If user clicks Skip:                                                       │
│  → Send: { feedback_type: "thumbs_down" }  (no additional detail)          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Feedback Reasons (Tags)

| UI Option | Tag Sent | Priority |
|-----------|----------|----------|
| "This is incorrect" | `factually_wrong` | High - routes to domain expert |
| "This is outdated" | `outdated` | Medium - triggers freshness check |
| "This didn't answer my question" | `incomplete` | Medium - RAG/synthesis issue |
| "Other" | `other` | Normal |

#### Changes Required in qa-bot-core

**File**: `src/utils/flows/qa-flow.tsx`

1. **Add feedback state machine**: Instead of immediately sending on 👎 click, transition to `feedback_detail` state

2. **New flow states**:
   ```
   qa_loop → (👎) → feedback_detail → (submit) → qa_loop
                                    → (skip)   → qa_loop
   ```

3. **Expanded payload**: Change from `{ rating: 0 }` to:
   ```json
   {
     "sessionId": "uuid",
     "queryId": "uuid",
     "feedback_type": "thumbs_down",
     "tags": ["factually_wrong"],
     "comment": "optional user comment"
   }
   ```

4. **Backward compatibility**: If no `ratingEndpoint` configured, skip feedback entirely (current behavior)

#### Component Props

New optional props for QABot component:

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `expandedFeedback` | boolean | `true` | Show reason selection on thumbs-down |
| `feedbackReasons` | array | (see above) | Customize available reasons |
| `requireFeedbackReason` | boolean | `false` | Require reason selection (no Skip) |

### Agent Implementation

**Repository**: `access-agent` (LangGraph Python)

The agent receives feedback from the chat UI and routes it to Argilla and Grafana.

#### New Endpoint

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/feedback` | POST | Receive feedback from chat UI |

#### Responsibilities

1. **Receive feedback** from chat UI (session_id, query_id, feedback_type, tags, comment)
2. **Retrieve Q&A context** for the query being rated (question, response, tools used, RAG matches)
3. **Log to Grafana** - all feedback (positive and negative) for metrics
4. **Push to Argilla** - negative feedback only, for human review

#### Context Retrieval

The agent must retrieve the original Q&A to include in the Argilla record. Options:

| Approach | Pros | Cons |
|----------|------|------|
| **LangGraph checkpointer** | Already stores conversation state | Couples feedback to checkpointer internals |
| **Redis cache** | Simple, decoupled, TTL-based expiry | Additional infrastructure |

**Recommendation**: Cache Q&A context in Redis when response is sent (1 hour TTL). Retrieve on feedback submission.

#### Priority Assignment

Agent auto-assigns priority based on feedback tags:

| Tag | Priority | Rationale |
|-----|----------|-----------|
| `factually_wrong` | High | Incorrect info needs urgent review |
| `outdated` | Normal | May need source data refresh |
| `incomplete` | Normal | RAG/synthesis tuning |
| `other` | Normal | Needs triage |
| (no tags) | Low | Less actionable without detail |

### Argilla Dataset Changes

**Dataset**: `feedback-review` (in `production` workspace)

#### New/Updated Fields

| Field | Type | New? | Description |
|-------|------|------|-------------|
| `feedback_tags` | text[] | **New** | User-selected reasons from UI |
| `user_comment` | text | **New** | Optional free-text from user |
| `rag_matches` | json | **New** | RAG results for debugging |
| `trace_id` | string | **New** | Links to Grafana traces |
| `priority` | enum | **New** | Auto-assigned: high/normal/low |

#### New Filters

Reviewers can filter the queue by:
- Priority (high first)
- Feedback tags (e.g., show only `factually_wrong`)
- Tools used (route to domain experts)
- Date range

#### Review Workflow Changes

1. **Priority sorting**: High priority items surface first in review queue
2. **Tag visibility**: Feedback tags and user comment shown prominently to reviewer
3. **Trace link**: Reviewer can click trace_id to see full conversation in Grafana

### Future: Integration with Researcher Profile

If the user has an [AI profile enabled](./09-researcher-profiles.md), feedback can inform personalization:

| Feedback | Profile Update |
|----------|----------------|
| Repeated thumbs-down on GPU recommendations | Note: user finds GPU info unhelpful |
| Thumbs-up on detailed explanations | Preference: detailed responses |
| Flag "outdated" on allocation info | Note: user sensitive to stale data |

This is **opt-in only** and requires the Drupal profile feature (Phase 2+).

---

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

1. **Reviewer onboarding**: Who are the initial reviewers for each domain?

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
