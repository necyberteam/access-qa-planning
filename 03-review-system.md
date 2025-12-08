# Review System

> **Part of the Data Pipeline**: [Training Data](./02-training-data.md) → This doc → [Model Training](./04-model-training.md)
>
> **Related**: [Agent Architecture](./01-agent-architecture.md)

## Overview

Web application for reviewing Q&A data at two critical points:

1. **Pre-Training Review**: Approve Q&A pairs before they go into training data
2. **Post-Deployment Feedback**: Capture user feedback on model outputs, flag bad answers for correction

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
│  User Q&A DB ─────┼──▶ Review UI ──▶ Approved    │                         │
│                   │    (approve/   Training  ──▶ Model ──▶ Response         │
│  Documentation ───┘    reject)     Data          │                         │
│                                                  ▼                         │
│                                           User feedback                     │
│                                           (thumbs up/down)                  │
│                                                  │                         │
│                                                  ▼                         │
│                                           Review UI ◀──── Flag bad answers │
│                                           (correct & retrain)              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## User Roles

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Reviewer** | View, Approve, Reject, Edit own domain | Domain experts review their area |
| **Admin** | All permissions + user management | Oversee entire pipeline |
| **Viewer** | Read-only access to approved data | Stakeholders checking progress |

---

## Core Features

### 1. Review Queue Dashboard

- Pending counts by domain (compute-resources, software-discovery, allocations, etc.)
- Recent activity feed (extractions, approvals, imports)
- Training data status (total approved, pending, delta since last training)
- Quick actions: Trigger retraining, Export data, View statistics

### 2. Review Interface

For each Q&A pair:
- **Source info**: Where it came from (MCP extraction, user Q&A, doc-generated)
- **Question**: Editable
- **Answer**: Editable, with markdown preview
- **Citation**: Verify `<<SRC:...>>` markers are correct
- **Source data viewer**: See the raw data the Q&A was derived from
- **Similar Q&A detection**: Alert if duplicates exist with similarity scores

Actions:
- Approve
- Reject (with reason: duplicate, incorrect, vague, incomplete, citation issue)
- Flag for discussion
- Skip

### 3. Duplicate Handling

When similar questions detected:
- Side-by-side comparison view
- Decision options: Keep current, Keep existing, Merge, Keep both
- Similarity threshold: ~0.85 for flagging

### 4. Source Data Viewer

For MCP-extracted Q&A:
- Show raw API data the Q&A was generated from
- Highlight changes since last extraction
- Link to all Q&A pairs using this source
- Option to regenerate Q&A for changed fields

### 5. Post-Deployment Feedback Queue

Captures feedback from production:

**Explicit User Feedback**:
- Thumbs up/down from QA Bot interface
- Negative feedback automatically queued for review
- Positive feedback used to reinforce good patterns

**Implicit Feedback Signals**:
- Follow-up questions (may indicate incomplete answer)
- Session abandonment after response (possible dissatisfaction)
- Copy/paste of response content (positive signal)
- Time spent reading response (engagement metric)
- Re-asking similar question (possible confusion)

These signals are logged for pattern analysis, not individual review.

**Drift Detection**:
- Periodic comparison of model answers vs live MCP data
- Flagged when model gives outdated information
- Triggers retrain consideration

**Review Actions for Flagged Responses**:
- Correct the answer → add to training data
- Mark as acceptable → dismiss flag
- Identify pattern → create new training examples

---

## Notification System

### Event Types

| Event | Recipients | Channel | Priority |
|-------|------------|---------|----------|
| New items pending review | Domain reviewers | Email + Slack | Normal |
| Review threshold reached (>N pending for >3 days) | Assigned reviewers | Slack | Normal |
| Extraction failed | Admins | Email + Slack | High |
| Retrain threshold reached | Admins | Email + Slack | High |
| Weekly summary | All users | Email | Low |

### User Preferences

Per user:
- Email notification frequency (immediate / daily digest)
- Slack @mentions
- Which domains they review

---

## Tech Stack

### Frontend
- **Framework**: Nuxt 3 (Vue.js)
- **UI Library**: Nuxt UI Pro (dashboard layouts, tables, forms)
- **Header/Footer**: access-ci-ui web components
- **Markdown**: For answer preview

### Backend
- **Framework**: Nuxt 3 server routes (`/server/api/`)
- **Database**: PostgreSQL
- **ORM**: Prisma or Drizzle
- **Queue**: Redis + BullMQ for background jobs (extractions, notifications)

### Authentication

**CILogon + ACCESS IdP**

```
User → CILogon → ACCESS IdP → OAuth tokens → Nuxt session
```

Configuration:
- Force ACCESS IdP selection via `idp_hint`
- User info from CILogon: sub, name, email, idp, affiliation

### Notifications
- **Slack**: @slack/web-api for bot messages
- **Email**: Mailgun via nodemailer-mailgun-transport
- **In-app**: Server-Sent Events or WebSocket

---

## Data Model

### Core Tables

**qa_pairs**
- id, source (mcp_extraction | user_qa | doc_generated), domain
- question, answer, citation_ids (JSONB), metadata (JSONB)
- status (pending | approved | rejected | flagged), complexity
- created_at, updated_at

**reviews**
- qa_pair_id, reviewer_id, action, rejection_reason
- previous_content (if edited), notes, created_at

**source_data**
- id, domain, data (JSONB), extracted_at
- previous_hash, current_hash, changes (JSONB diff)

**duplicate_clusters**
- qa_pair_ids (JSONB array), similarity_scores
- resolved, resolution (kept_first | kept_second | merged | kept_both)

**notifications**
- user_id, type, title, message, link, read, created_at

**training_runs**
- version, started_at, completed_at, status
- qa_pair_count, config (JSONB), metrics (JSONB)

### Key Indexes
- qa_pairs: status, domain, source
- notifications: user_id + read

---

## API Endpoints

### Q&A Management
- `GET /api/qa-pairs` - List with filters
- `GET /api/qa-pairs/:id` - Get single pair
- `PUT /api/qa-pairs/:id` - Update pair
- `POST /api/qa-pairs/:id/approve` - Approve
- `POST /api/qa-pairs/:id/reject` - Reject
- `POST /api/qa-pairs/:id/flag` - Flag for discussion

### Review Queue
- `GET /api/review/queue` - Pending items
- `GET /api/review/queue/:domain` - Pending for domain
- `GET /api/review/stats` - Statistics

### Training
- `GET /api/training/status` - Current status
- `POST /api/training/trigger` - Trigger retraining
- `GET /api/training/export` - Export training data

---

## Implementation Phases

### Phase 1: Core Review Interface (MVP)
- Basic dashboard with pending counts
- Review interface (approve/reject)
- Simple edit capability
- PostgreSQL storage

### Phase 2: Source Data Integration
- Source data viewer
- Change detection display
- Link Q&A to source entities

### Phase 3: Duplicate Detection
- Similarity scoring (embeddings)
- Duplicate cluster view
- Merge/resolve workflow

### Phase 4: Notifications
- Email notifications (Mailgun)
- Slack integration
- In-app notification center
- User preferences

### Phase 5: Analytics & Admin
- Review statistics and dashboards
- Training run management
- User management
- Audit log

---

## Open Questions

1. **Initial reviewers**: Who are the first users/domain experts?
2. **SLA**: Expected review turnaround time?
3. **Hosting**: Where will this be deployed within ACCESS infrastructure?

→ *Next in pipeline: [Model Training](./04-model-training.md)* - What happens with approved data
