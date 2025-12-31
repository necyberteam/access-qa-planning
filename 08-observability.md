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

### Agent (Python)

| Location | Span Name | Attributes |
|----------|-----------|------------|
| `/query` endpoint | `http.request` | `request_id`, `session_id` |
| `run_agent()` | `agent.run` | `question_id` |
| `plan_node()` | `agent.plan` | `requires_tools`, `tool_count` |
| `execute_node()` | `agent.execute` | `strategy`, `tool_count` |
| `MCPClient.call_tool()` | `mcp.call_tool` | `server`, `tool_name`, `success`, `duration_ms` |
| `synthesize_node()` | `agent.synthesize` | `answer_length` |

### MCP Servers (Node.js)

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

## Implementation Phases

### Phase 1: Grafana Cloud Setup

- Create Grafana Cloud account (free tier)
- Note OTLP endpoint and API key
- Create service account for agent/MCP servers

**Deliverable:** Working Grafana Cloud instance with credentials

---

### Phase 2: Agent Instrumentation (Python)

- Add OpenTelemetry dependencies
- Create telemetry setup module
- Instrument API routes
- Instrument agent nodes
- Instrument MCP client
- Add environment variables for Grafana credentials
- Test trace export locally

**Deliverable:** Agent sends traces to Grafana Cloud

---

### Phase 3: MCP Server Instrumentation (Node.js)

- Add OpenTelemetry dependencies to shared package
- Create telemetry setup in shared package
- Initialize telemetry in each MCP server
- Add custom spans for tool execution
- Propagate trace context from incoming requests
- Test trace export

**Deliverable:** MCP servers send traces to Grafana Cloud, correlated with agent traces

---

### Phase 4: Request ID Propagation

- Generate `X-Request-ID` in qa-bot-core if not present
- Accept `X-Request-ID` in agent API
- Pass `X-Request-ID` to MCP servers
- Log `X-Request-ID` in all services

**Deliverable:** Single request ID visible across all services

---

### Phase 5: Dashboards

- Create System Overview dashboard
- Create User Activity dashboard
- Create MCP Tools dashboard
- Set up saved searches in trace explorer
- Configure dashboard permissions

**Deliverable:** Stakeholders can view system activity via web UI

---

### Phase 6: Alerting (Optional)

- Configure notification channels (Slack, email)
- Create alert rules
- Test alerting

**Deliverable:** Automated notifications for errors and anomalies

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
