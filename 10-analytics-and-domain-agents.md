# Analytics Reporting & Domain Agents

> **Related**: [Observability](./08-observability.md) | [Agent Architecture](./01-agent-architecture.md) | [Events Actions](./05-events-actions.md) | [Capability Registry](./11-capability-registry.md)

## Overview

Two major features in progress for the ACCESS QA system:

1. **Analytics Reporting** — Automated weekly/monthly reports combining GA4 chatbot UI metrics with PostgreSQL agent usage data, delivered via email (Mailgun)
2. **Domain Agent Routing** — Specialized react agents for management operations (announcements, tickets) that bypass the general RAG pipeline

---

## Part 1: Analytics Reporting

### Architecture

```
[GA4 Property 326843153]          [PostgreSQL usage_logs]
  (chatbot UI events)                (agent metrics)
         │                                │
         ▼                                ▼
   ┌──────────────┐              ┌──────────────────┐
   │  ga4_client   │              │   db_reports      │
   │  (GA4 Data    │              │   (SQLAlchemy)    │
   │   API v1beta) │              │                   │
   └──────┬───────┘              └─────────┬─────────┘
          │                                │
          └────────────┬───────────────────┘
                       ▼
              ┌──────────────────┐
              │ unified_report    │
              │ (markdown render) │
              └────────┬─────────┘
                       │
              ┌────────┴─────────┐
              ▼                  ▼
     ┌──────────────┐   ┌──────────────┐
     │  Mailgun API  │   │  Slack       │
     │  (email)      │   │  (webhook)   │
     └──────────────┘   └──────────────┘
```

### Implementation Status

| Component | Status | Location |
|-----------|--------|----------|
| GA4 client | Done | `access-agent/src/reports/ga4_client.py` |
| DB reporter | Done | `access-agent/src/reports/db_reports.py` |
| Unified report | Done | `access-agent/src/reports/unified_report.py` |
| Mailgun delivery | Done | `access-agent/src/reports/deliver.py` |
| Slack delivery | Done (code) | Not configured yet (no webhook URL) |
| CLI entrypoint | Done | `access-agent/src/reports/__main__.py` |
| Docker compose config | Done | `docker-compose.prod.yml` — env vars + GA4 key mount |
| Committed to main | Done | `605fb83`, expanded in later commits |
| Deployed to server | Done | Pushed, .env configured, GA4 key mounted |
| Report scheduling | Done | Cron installed by deploy workflow (Monday 9am ET / 14:00 UTC) |

### CLI Usage

```bash
python -m src.reports weekly                    # Print to stdout
python -m src.reports weekly --email            # Send via Mailgun
python -m src.reports weekly --slack            # Send via Slack
python -m src.reports monthly                   # 30-day lookback
python -m src.reports --days 14 --output report.md
python -m src.reports weekly --start 2026-01-01 --end 2026-01-31
python -m src.reports weekly --no-ga4           # Skip GA4 (agent metrics only)
```

### Environment Variables

```env
# GA4 (Google Analytics 4 Data API)
GA4_PROPERTY_ID=326843153
GA4_CREDENTIALS_FILE=mghpcc-4626301de758.json

# Email (Mailgun HTTP API)
MAILGUN_API_KEY=<sending key>
MAILGUN_DOMAIN=mg.sweetandfizzy.com
REPORT_EMAIL_FROM=reports@mg.sweetandfizzy.com
REPORT_EMAIL_TO=andrew@elytra.net

# Slack (optional)
REPORT_SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz
```

### GA4 Chatbot Events

Events tracked via GTM with regex trigger `chatbot_.*`. Two layers:

**Core events** (fired by `qa-bot-core`, enriched with `timestamp`, `sessionId`, `pageUrl`, `isEmbedded`):

| Event | Description |
|-------|-------------|
| `chatbot_open` | User opens the chatbot |
| `chatbot_close` | User closes chatbot (includes `messageCount`, `durationMs`) |
| `chatbot_new_chat` | User starts a new conversation |
| `chatbot_question_sent` | User submits a question (general Q&A) |
| `chatbot_answer_received` | Bot returns a response (includes `responseTimeMs`, `success`) |
| `chatbot_answer_error` | API error on response |
| `chatbot_login_prompt_shown` | Login gate displayed to unauthenticated user |
| `chatbot_rating_sent` | User rates a response (`helpful` / `not_helpful`) |

**ACCESS layer events** (fired by `access-qa-bot` flows):

| Event | Description |
|-------|-------------|
| `chatbot_menu_selected` | User selects a menu option |
| `chatbot_ticket_started` | Ticket flow initiated |
| `chatbot_ticket_submitted` | Ticket successfully created |
| `chatbot_ticket_error` | Ticket creation failed |
| `chatbot_ticket_step` | Progress through ticket steps |
| `chatbot_security_started` | Security report flow initiated |
| `chatbot_security_submitted` | Security report submitted |
| `chatbot_metrics_question_sent` | XDMoD metrics question asked |
| `chatbot_file_uploaded` | File attached to ticket |

**Custom dimensions** (registered in GA4 Admin > Custom Definitions):
- `customEvent:selection` — Menu choice tracking (DLV configured in GTM)
- `customEvent:ticketType` — Ticket type breakdown (DLV configured in GTM)
- `customEvent:isEmbedded` — Embedded vs floating widget mode (DLV added in GTM; pending GA4 registration — need 24-48h for parameter to appear after first event)

**Built-in dimensions** used by reports (no registration needed):
- `pagePath` — Which page the chatbot was used on

**Environment dimension**: `chatbot_env` passed with every event (values: `live`, `dev`, `test`, `local`, multidev names). Requires:
- GTM: Add `chatbot_env` as event parameter in the GA4 tag + create DLV variable
- GA4: Register `chatbot_env` as custom dimension

### GA4 dataLayer Fix

**Problem**: The Drupal `headerfooter.js` was pushing events to `window.dataLayer` without the `event` key that GTM requires.

**Fix** (deployed Feb 19, 2026):
```js
// Before (broken — GTM didn't recognize events):
window.dataLayer.push(event);

// After (working — GTM matches on `event` key):
window.dataLayer.push({ event: event.type, ...event, chatbot_env: chatbotEnv });
```

**Location**: `access/js/headerfooter.js` line 106
**Environment source**: `drupalSettings.access.environment` set from `PANTHEON_ENVIRONMENT` in `access.module`

### Agent Metrics (PostgreSQL)

The `usage_logs` table captures per-query metrics:

| Column | Type | Purpose |
|--------|------|---------|
| timestamp | datetime | When query was logged |
| session_id | string | Conversation session |
| question_id | string | Individual question |
| query_text | text | User's question |
| query_type | string | static/dynamic/combined |
| topic | string | Classification |
| confidence | string | Classification confidence |
| tools_used | JSONB array | MCP tools called |
| duration_ms | float | Execution time |
| user_hash | string | SHA256 hash of user ID (no PII) |
| response_length | int | Response character count |
| success | string | true/false |

### Report Sections

**GA4 (Chatbot UI):**
- Sessions and unique users
- Engagement: opens, new chats
- Login prompts shown (if any)
- Ticket completion metrics (started → submitted, completion rate, errors)
- Security report counts
- AI questions (general): asked → answered | errors (from `chatbot_question_sent`/`chatbot_answer_received`/`chatbot_answer_error`)
- AI questions (XDMoD): from `chatbot_metrics_question_sent`
- Feedback ratings (if any)
- Menu selections breakdown
- Ticket types breakdown
- Top pages: top 10 pages by event count with percentages (from `pagePath` dimension)
- Embedded vs floating: breakdown by widget mode (requires `isEmbedded` custom dimension in GA4; omitted gracefully if not registered)

**Agent (PostgreSQL):**
- Total queries, unique users, success rate
- Response time percentiles (p50, p95, p99, avg)
- Query type distribution with percentage bars
- Top 10 topics
- Top 10 MCP tools used
- Daily trend table
- Content gaps (low confidence / failed queries by topic with sample queries)

### Remaining Work

1. ~~Push commit to deploy~~ — Done
2. ~~Server .env~~ — Done (Mailgun + GA4 vars configured)
3. ~~GA4 key file~~ — Done (mounted in container)
4. ~~GTM DLVs~~ — `selection`, `ticketType`, `isEmbedded` configured
5. **GA4 custom dimension: `isEmbedded`** — DLV added in GTM; waiting 24-48h for parameter to appear in GA4 Admin after first event, then register
6. **GA4 custom dimension: `chatbot_env`** — GTM DLV + GA4 registration still needed
7. **Slack webhook** — Create and configure when ready

**Scheduling**: The deploy workflow automatically installs a cron job:
- `scripts/send-weekly-report.sh` — runs `python -m src.reports weekly --email` inside the container
- Cron: `0 14 * * 1` (Monday 9am ET / 14:00 UTC)
- Output logged via `logger -t access-reports` to syslog

---

## Part 2: Domain Agent Routing

### Concept

A specialized routing system where management operations (create/update/delete) are handled by domain-specific react agents with direct MCP tool access, bypassing the general RAG pipeline.

```
User: "Create an announcement about maintenance"

START → classify (domain=announcements)
     → domain_agent (react loop with announcements MCP tools)
     → END

User: "What GPUs does Delta have?"

START → classify (domain=null)
     → rag_answer → plan → execute → synthesize
     → END
```

### Implementation Status

| Component | Status | Location |
|-----------|--------|----------|
| Domain config system | Done | `src/agent/domains/config.py` |
| Domain registry | Done | `src/agent/domains/registry.py` |
| Announcements domain | Done | `src/agent/domains/announcements.py` |
| JSM domain | Done | `src/agent/domains/jsm.py` |
| MCP tool wrapping | Done | `src/agent/domains/tools.py` |
| Domain agent node | Done | `src/agent/nodes/domain_agent.py` |
| Classifier extension | Done | `src/agent/nodes/classify.py` (adds `domain` field) |
| Graph routing | Done | `src/agent/graph.py` (domain_agent → END, or fallback to UKY if no tools) |
| State extension | Done | `src/agent/state.py` (QueryClassification.domain) |
| **Committed** | **Yes** | PR #1 (`uky-plus-mcp` branch, 14 commits off main) |
| **Tests** | **Yes** | 18-question battery: 18/18 pass, domain routing validated end-to-end |
| **MCP acting user** | **Resolved** | `MCP_API_KEY` shared secret enables service-to-service auth |

### Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        DOMAIN AGENT ROUTING                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────┐                                                    │
│   │  classify    │ ─── domain = "announcements" ──┐                  │
│   │  (extended)  │ ─── domain = "jsm" ────────────┤                  │
│   │             │ ─── domain = null ──────────────┼──▶ RAG pipeline  │
│   └─────────────┘                                 │                  │
│                                                   ▼                  │
│                                     ┌──────────────────────┐         │
│                                     │   domain_agent node   │         │
│                                     │                       │         │
│                                     │  1. Look up config    │         │
│                                     │  2. Create MCP tools  │         │
│                                     │  3. React agent loop  │         │
│                                     │  4. Return response   │         │
│                                     └──────────┬───────────┘         │
│                                                │                     │
│                                                ▼                     │
│                                              END                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Domain Definitions

**Announcements** (`src/agent/domains/announcements.py`):
- MCP servers: `["announcements"]`
- Max iterations: 15
- Operations: Create, update, delete announcements
- Workflow: Gather context → preview → confirm → create → return edit URL

**JSM — Jira Service Management** (`src/agent/domains/jsm.py`):
- MCP servers: `["jsm"]`
- Max iterations: 15
- Operations: Create tickets, report issues, file security incidents
- Workflow: Classify ticket type → gather fields conversationally → submit

### MCP Tool Wrapping

`src/agent/domains/tools.py` wraps MCP tools for use with LangChain's `create_react_agent`:

1. `_build_args_schema()` — Dynamically creates pydantic models from MCP tool JSON Schema params
2. `MCPToolWrapper` — Extends `BaseTool`, calls `MCPClient.call_tool()` with `X-Acting-User`
3. `create_domain_tools()` — Factory that filters catalog by domain's MCP servers and wraps each tool

### Classification Extension

The classifier prompt now detects domain intent:

| domain value | When detected |
|-------------|---------------|
| `"announcements"` | User wants to CREATE/UPDATE/DELETE/MANAGE announcements |
| `"jsm"` | User explicitly asks to OPEN/FILE a ticket, REPORT an issue. Problem descriptions and complaints route to RAG, not JSM — only explicit ticket-filing language triggers domain routing. |
| `null` | Searches, informational queries, general questions, problem descriptions |

### Supporting Changes

| File | Change |
|------|--------|
| `src/agent/nodes/plan.py` | Added instruction for tool ordering prerequisites |
| `src/agent/nodes/synthesize.py` | Simplified prompts for clarity |
| `src/tools/mcp_client.py` | Added API key header support for servers requiring auth |
| `src/main.py` | CORS respects `ALLOWED_ORIGINS` env var |

### Resolved Dependencies

1. **~~MCP Acting User Gap~~** — Resolved via `MCP_API_KEY` shared secret for service-to-service auth. Agent sends API key in header; MCP servers authenticate the agent as a trusted service. Per-user attribution via `X-Acting-User` header works for announcements and JSM.

2. **~~Username Resolution~~** — MCP servers resolve ACCESS ID to Drupal UID via JSON:API.

### Resolved Issues (PR #1)

1. **JSM error handling** — When JSM server is unavailable (no tools), domain agent returns `final_answer=None` and clears `domain` to fall back to UKY RAG answer instead of propagating errors.
2. **JSM misrouting** — Tightened classification to require explicit ticket-filing language. Problem descriptions ("I can't log in to Bridges-2") now route to RAG for troubleshooting, not JSM for ticket creation.
3. **Domain agent committed** — All domain agent code is committed on `uky-plus-mcp` branch (PR #1, 14 commits).
4. **End-to-end testing** — 18-question battery includes Q17 (JSM) and Q18 (Announcements), both pass.

### Remaining Work

1. **Merge PR #1** into main
2. **Message history management** — React loop currently passes full conversation; may need context window limits for long sessions
3. **Capability registry integration** — Attach capability definitions to domain configs (see [11-capability-registry.md](./11-capability-registry.md))

---

## Part 3: Observability Status Update

See [08-observability.md](./08-observability.md) for full plan. Current status:

| Phase | Status | Notes |
|-------|--------|-------|
| Phase 1: Grafana Cloud setup | Done | Using Honeycomb instead of Grafana (OTLP endpoint) |
| Phase 2: Agent instrumentation (Python) | Done | OpenTelemetry SDK, FastAPI/HTTPX/LangChain auto-instrumentation |
| Phase 3: MCP server instrumentation (Node.js) | Not started | |
| Phase 4: Request ID propagation | Partial | session_id + question_id tracked; full X-Request-ID not yet |
| Phase 5: Dashboards | Not started | Using Honeycomb Explore for now |
| Phase 6: Alerting | Not started | |
| **Analytics reports** | **Done** | `src/reports/` — GA4 + PostgreSQL + Mailgun delivery |

**Note**: Observability doc references Grafana Cloud but we're actually using Honeycomb (`OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io`). The doc should be updated to reflect this.

---

## Priority Order

1. ~~Deploy analytics reports~~ — Done (deployed, cron scheduled, first report sent)
2. ~~MCP acting user~~ — Resolved via MCP_API_KEY service auth
3. ~~Domain agent testing~~ — Committed on PR #1, 18/18 battery pass
4. **GA4 custom dimensions** — Register `isEmbedded` and `chatbot_env` in GA4 Admin
5. **Merge PR #1** — 14 commits on `uky-plus-mcp`, ready for main
6. **Capability registry** — Next major feature (see [11-capability-registry.md](./11-capability-registry.md))
7. **Observability improvements** — MCP instrumentation, dashboards, alerting
