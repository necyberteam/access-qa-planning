# Observability

> **Related**: [Backend Integration Spec](./07-backend-integration-spec.md) | [Agent Architecture](./01-agent-architecture.md)

## Overview

Enable distributed tracing and operational monitoring across the ACCESS QA system to support:

1. **Debugging** - Trace requests across agent → MCP servers → backends
2. **Monitoring** - Dashboards for system health and performance
3. **Alerting** - Notify on errors, latency spikes, unusual patterns

**Note:** User-level auditing ("who did what") is handled by backend systems (Drupal, etc.), not Grafana. See [Privacy Considerations](#privacy-considerations).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           OBSERVABILITY ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│   │ qa-bot   │     │  agent   │     │   MCP    │     │ backends │              │
│   │ frontend │────▶│ (Python) │────▶│ servers  │────▶│ (Drupal, │              │
│   │          │     │          │     │  (Node)  │     │  etc.)   │              │
│   └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘              │
│        │                │                │                │                     │
│        │         X-Request-ID + X-Acting-User             │                     │
│        │         ◀──────────────────────────────────────▶ │                     │
│        │                │                │                │                     │
│        │                ▼                ▼                │                     │
│        │         ┌─────────────────────────────┐          │                     │
│        │         │     OpenTelemetry SDK       │          │                     │
│        │         │  (traces, logs, metrics)    │          │                     │
│        │         └─────────────┬───────────────┘          │                     │
│        │                       │ OTLP                     │                     │
│        │                       ▼                          │                     │
│        │         ┌─────────────────────────────┐          │                     │
│        │         │      Grafana Cloud          │          │                     │
│        │         │  ┌───────┐ ┌──────┐ ┌─────┐ │          │                     │
│        │         │  │ Tempo │ │ Loki │ │Prom │ │          │                     │
│        │         │  │traces │ │ logs │ │metrics│          │                     │
│        │         │  └───────┘ └──────┘ └─────┘ │          │                     │
│        │         └─────────────┬───────────────┘          │                     │
│        │                       │                          │                     │
│        │                       ▼                          │                     │
│        │         ┌─────────────────────────────┐          │                     │
│        │         │       Dashboards            │◀─────────┘                     │
│        │         │   (stakeholder access)      │                                │
│        │         └─────────────────────────────┘                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Grafana Cloud

### Why Grafana Cloud

| Factor | Benefit |
|--------|---------|
| **Free tier** | 50GB logs, 50GB traces/month - sufficient for initial deployment |
| **OpenTelemetry native** | Direct OTLP ingestion, no collector infrastructure needed |
| **Multi-service correlation** | Single view across Python agent and Node MCP servers |
| **Stakeholder access** | Web UI with dashboards, role-based access control |
| **Managed** | No infrastructure to maintain |

### Components Used

| Component | Purpose | Data |
|-----------|---------|------|
| **Tempo** | Distributed tracing | Request flows, latency, errors |
| **Loki** | Log aggregation | Structured logs with trace correlation |
| **Prometheus** | Metrics (optional) | Request counts, latency histograms |
| **Grafana** | Visualization | Dashboards, alerting |

---

## Request Correlation

### Headers

Every request through the system carries correlation headers:

| Header | Purpose | Format | Set By |
|--------|---------|--------|--------|
| `X-Request-ID` | Correlate logs/traces across services | UUID v4 | Frontend or first service |
| `X-Acting-User` | Authorization at backends | `username@access-ci.org` | Frontend (from auth) |

**Note:** `X-Acting-User` is passed through for backend authorization (e.g., Drupal needs to know who to attribute content to) but is **not logged** to Grafana. See [Privacy Considerations](#privacy-considerations).

### Propagation

```
Frontend                    Agent                      MCP Server                 Backend
    │                         │                            │                        │
    │  X-Request-ID: abc123   │                            │                        │
    │  X-Acting-User: jsmith  │  (passed through,          │                        │
    │────────────────────────▶│   not logged)              │                        │
    │                         │                            │                        │
    │                         │  X-Request-ID: abc123      │                        │
    │                         │  X-Acting-User: jsmith     │                        │
    │                         │───────────────────────────▶│                        │
    │                         │                            │                        │
    │                         │                            │  X-Request-ID: abc123  │
    │                         │                            │  X-Acting-User: jsmith │
    │                         │                            │───────────────────────▶│
    │                         │                            │                        │
    │                         │                            │    (backend stores     │
    │                         │                            │     audit trail)       │
```

---

## Instrumentation Points

### Agent (Python) - Implemented

| Location | Span Name | Attributes |
|----------|-----------|------------|
| `run_agent()` | `agent.run` | `query`, `session_id`, `question_id`, `user`, `tools_used`, `answer_length` |
| `classify_node()` | `agent.classify` | `query_type`, `confidence`, `reason` |
| `rag_answer_node()` | `agent.rag_answer` | `threshold`, `matches_found`, `best_score`, `result` |
| `plan_node()` | `agent.plan` | `query`, `tools_planned`, `strategy`, `requires_tools` |
| `execute_node()` | `agent.execute` | `tools_executed`, `tools_succeeded` |
| `synthesize_node()` | `agent.synthesize` | `strategy`, `has_rag`, `has_tools`, `answer_length` |
| `MCPClient.call_tool()` | `mcp.call_tool.{tool}` | `server`, `tool`, `arguments`, `success`, `result.*` |
| `QAServiceClient.search()` | `rag.search` | `duration_ms`, `cached`, `matches_returned` |

### LLM Calls (Auto-Instrumented via LangChain)

| Location | Span Name | Attributes |
|----------|-----------|------------|
| Any `ChatOpenAI.ainvoke()` | `ChatOpenAI.chat` | `langgraph_node`, `ls_model_name`, `ls_temperature`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_read_input_tokens` |

### MCP Servers (Node.js) - Not Yet Implemented

| Location | Span Name | Attributes |
|----------|-----------|------------|
| Tool handler entry | `tool.{tool_name}` | `request_id`, tool-specific args (no user data) |
| Backend API call | `backend.{service}` | `method`, `endpoint`, `status` |

---

## Dashboards

### System Overview

**Audience:** Operations, developers

| Panel | Description |
|-------|-------------|
| Request rate | Queries per minute |
| Error rate | Percentage of failed queries |
| Latency percentiles | P50, P95, P99 response times |
| Active sessions | Unique `session_id` values in last hour |
| Tool usage | Breakdown by tool name |

### MCP Tools

**Audience:** Developers

| Panel | Description |
|-------|-------------|
| Tool call rate | Calls per minute by tool |
| Tool latency | Response time by server/tool |
| Tool errors | Error rate by tool |
| Slowest tools | P95 latency ranking |

### Trace Explorer

**Audience:** Developers

- Search by `request_id`, `session_id`, or time range
- Drill down into individual traces
- View span waterfall across services

---

## Alerting

| Alert | Condition | Notification |
|-------|-----------|--------------|
| High error rate | >5% errors in 5 minutes | Slack + email |
| High latency | P95 > 10s for 5 minutes | Slack + email |
| MCP server down | Health check fails | Slack + email |
| Unusual activity | 3x normal query rate | Slack (info only) |

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `GRAFANA_OTLP_ENDPOINT` | OTLP HTTP endpoint | `https://otlp-gateway-prod-us-central-0.grafana.net/otlp` |
| `GRAFANA_OTLP_TOKEN` | Base64-encoded `instance_id:api_key` | `MTIzNDU2OmdsY18...` |
| `OTEL_SERVICE_NAME` | Service name for traces | `access-agent` |

---

## Cost Considerations

**Grafana Cloud Free Tier:**
- 50 GB logs/month
- 50 GB traces/month
- 10,000 metrics series
- 14-day retention

**Estimated usage:**
- ~1KB per trace (typical query with 3-4 tools)
- ~500 bytes per log line
- At 1000 queries/day: ~30MB traces/day, ~1GB/month (well under limit)

**Scaling:**
- If usage exceeds free tier, Grafana Cloud pay-as-you-go is reasonable
- Alternative: self-host Grafana + Tempo + Loki (more ops overhead)

---

## Privacy Considerations

**No user identifiers are logged to Grafana.** This is intentional to comply with privacy requirements.

| Data | Logged? | Notes |
|------|---------|-------|
| `acting_user` | No | Passed through to backends for authorization, not logged |
| `session_id` | Yes | Anonymous UUID, not linked to user identity |
| `request_id` | Yes | Anonymous UUID for trace correlation |
| Query text | Yes (flagged) | Logged for debugging, but flagged if potential PII detected |
| Tool arguments | Partial | Only non-sensitive args (e.g., resource names, not user data) |

### Query Text PII Handling

Query text is valuable for debugging but may contain PII (e.g., "What's the status of my project TG-ABC123?" or "I'm John Smith and I need help").

**Approach:** Log query text but flag potential PII for filtering:

| Method | Description |
|--------|-------------|
| Pattern detection | Flag queries containing email addresses, project IDs, names |
| LLM classification | Use lightweight model to classify PII risk (low/medium/high) |
| Retention policy | Auto-delete flagged queries after shorter retention period |
| Dashboard filtering | Option to hide flagged queries from dashboards |

Flagged queries are still available for debugging specific issues but excluded from aggregate analysis and dashboards.

### Where Audit Data Lives

| Purpose | System | Contains User ID? |
|---------|--------|-------------------|
| Performance monitoring | Grafana | No |
| Request debugging | Grafana (traces) | No - use request_id + timestamp |
| Audit trail (who did what) | Backend systems (Drupal, etc.) | Yes - stored at source |

### Debugging Without User IDs

When a user reports an issue:
1. User provides approximate timestamp and what they asked about
2. Support searches Grafana by time window + characteristics (tool called, error type)
3. Session ID helps correlate multiple requests from same conversation

This approach provides operational visibility without collecting user PII in the observability stack.

---

## Security

- OTLP tokens stored as environment variables/secrets
- Dashboards require Grafana Cloud login
- Role-based access: viewers vs editors

---

## Implementation Status

### Phase 1: Grafana Cloud Setup ✅

- Grafana Cloud account created (accessci.grafana.net)
- OTLP endpoint configured for Tempo (traces)
- API tokens generated for agent access

---

### Phase 2: Agent Instrumentation (Python) ✅

Implemented in `access-agent/src/telemetry/`:

| Component | Status | Notes |
|-----------|--------|-------|
| OpenTelemetry SDK | ✅ | OTLP HTTP exporter to Grafana Cloud |
| FastAPI auto-instrumentation | ✅ | Via `opentelemetry-instrumentation-fastapi` |
| HTTPX auto-instrumentation | ✅ | Traces MCP HTTP calls |
| LangChain auto-instrumentation | ✅ | Via `opentelemetry-instrumentation-langchain` |
| Agent node spans | ✅ | classify, rag_answer, plan, execute, synthesize |
| MCP tool call spans | ✅ | Server, tool name, arguments, results |
| Root span linking | ✅ | All child spans under `agent.run` |

**LLM Call Tracing:**

LangChain instrumentation automatically captures for each LLM call:
- Model name (gpt-4o-mini, gpt-4o)
- Input/output token counts
- Cached token counts (OpenAI prompt caching)
- Latency

**CLI Tool:**

`scripts/traces.py` provides quick access to recent traces:

```bash
python scripts/traces.py           # Last 5 agent runs
python scripts/traces.py -n 10     # Last 10
python scripts/traces.py -t <id>   # Specific trace
python scripts/traces.py -m        # MCP calls only
```

---

### Phase 3: MCP Server Instrumentation (Node.js) 🚧

Not yet implemented. MCP servers currently don't export traces.

**TODO:**
- Add OpenTelemetry to `@access-mcp/shared`
- Instrument each MCP server
- Propagate trace context from agent

---

### Phase 4: Request ID Propagation 🚧

Partially implemented:
- Agent generates `question_id` for each query
- Session ID tracked across conversation turns
- **TODO:** Full X-Request-ID header propagation

---

### Phase 5: Dashboards 🚧

Not yet created. Using Grafana Explore for now.

**TODO:**
- System Overview dashboard
- MCP Tools dashboard
- LLM cost tracking dashboard

---

### Phase 6: Alerting 🚧

Not yet configured.

**TODO:**
- High error rate alerts
- Latency threshold alerts

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Trace coverage | 100% of requests traced |
| Correlation | Can find related logs/traces by `request_id` |
| Dashboard load time | <5s |
| Debugging effectiveness | Can diagnose issues from timestamp + session_id |

---

## References

- [Grafana Cloud OTLP](https://grafana.com/docs/grafana-cloud/send-data/otlp/)
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/)
- [OpenTelemetry Node.js](https://opentelemetry.io/docs/languages/js/)
- [Grafana Tempo](https://grafana.com/docs/tempo/latest/)
- [Grafana Loki](https://grafana.com/docs/loki/latest/)
