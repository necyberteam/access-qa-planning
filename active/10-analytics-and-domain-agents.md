# Analytics Reporting & Domain Agents

> **Related**: [Observability](./08-observability.md) | [Agent Architecture](./01-agent-architecture.md) | [Events Actions](./05-events-actions.md) | [Capability Registry](./11-capability-registry.md) | [Decision 005: Eval Pipeline](../decisions/005-llm-judge-eval-pipeline.md)

## Overview

Three reporting and agent capabilities:

1. **Analytics Reporting** (deployed) — Automated weekly reports combining GA4 chatbot UI metrics with PostgreSQL agent usage data, delivered via email (Mailgun) and Slack
2. **Domain Agent Routing** (deployed) — Specialized react agents for announcements CRUD and JSM ticket creation
3. **Quality Evaluation Reporting** (new) — Per-dimension quality scores from the eval pipeline (LLM judge + human review), with team/leadership/RP operator report formats. See [eval pipeline spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-31-eval-pipeline-design.md) and [eval README](https://github.com/necyberteam/access-agent/blob/main/eval/README.md)

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
| Graph routing | Done | `src/agent/graph.py` (domain_agent → END) |
| State extension | Done | `src/agent/state.py` (QueryClassification.domain) |
| **Committed** | **No** | All changes are uncommitted/unstaged |
| **Tests** | **No** | No unit or integration tests |
| **MCP acting user** | **Partial** | Agent sends `X-Acting-User` but MCP servers don't read it yet |

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
| `"jsm"` | User wants to CREATE a ticket, REPORT an issue, or FILE something |
| `null` | Searches, informational queries, general questions |

### Supporting Changes

| File | Change |
|------|--------|
| `src/agent/nodes/plan.py` | Added instruction for tool ordering prerequisites |
| `src/agent/nodes/synthesize.py` | Simplified prompts for clarity |
| `src/tools/mcp_client.py` | Added API key header support for servers requiring auth |
| `src/main.py` | CORS respects `ALLOWED_ORIGINS` env var |

### Blocking Dependencies

1. **MCP Acting User Gap** — Announcements (port 3009) and JSM (port 3012) servers need to read `X-Acting-User` header instead of static `ACTING_USER_UID` env var. Without this, the domain agent can't attribute actions to the correct user.

2. **Username Resolution** — MCP servers need to resolve ACCESS ID (e.g., `apasquale@access-ci.org`) to Drupal UID via JSON:API: `/jsonapi/user/user?filter[name]=username`

### Remaining Work

1. **Update MCP servers** to read `X-Acting-User` header (announcements, jsm)
2. **Test domain agent flows** end-to-end (announcements create, JSM ticket create)
3. **Add tests** — unit tests for classification, tool wrapping, and domain agent node
4. **Commit and deploy** domain agent changes
5. **Message history management** — React loop currently passes full conversation; may need context window limits
6. **Error UX** — What happens when MCP server is down or tool call fails mid-flow

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
2. **GA4 custom dimensions** — Register `isEmbedded` (after 24-48h) and `chatbot_env` in GA4 Admin
3. **MCP acting user** — Update announcements + JSM servers to read `X-Acting-User`
4. **Domain agent testing** — End-to-end test, then commit and deploy
5. **Observability improvements** — MCP instrumentation, dashboards, alerting
