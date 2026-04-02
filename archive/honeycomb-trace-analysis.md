# Honeycomb Trace Analysis Guide

This guide covers how to use Honeycomb for debugging, optimization, and understanding the ACCESS-CI agent system.

## Understanding Your Traces

Each query to the ACCESS-CI agent generates a trace showing the complete request flow:

```
HTTP Request (FastAPI)
└── agent.classify      # Determines if query needs tools
└── agent.plan          # Selects which MCP tools to call
└── agent.execute       # Runs the MCP tool calls
│   └── mcp.call_tool.search_resources
│   └── mcp.call_tool.get_infrastructure_news
└── agent.evaluate      # Checks if results are helpful
└── agent.synthesize    # Generates final response
```

## Reading the Waterfall Diagram

When viewing a trace in Honeycomb:

- **Horizontal bars** represent duration - longer bars indicate slower operations
- **Nesting** shows parent-child relationships between spans
- **Red/error indicators** highlight failures
- **Click any span** to view its attributes (tool arguments, response data, timing, etc.)

### Key Span Attributes

| Attribute | Description |
|-----------|-------------|
| `duration_ms` | How long the operation took |
| `name` | Operation name (e.g., `agent.execute`, `mcp.call_tool.search_resources`) |
| `mcp.tool` | The MCP tool being called |
| `mcp.server` | Which MCP server handled the request |
| `mcp.success` | Whether the MCP call succeeded |
| `session_id` | User's conversation session |
| `error` | Whether the span represents an error |

## Query Builder Basics

Honeycomb queries have four main components:

- **VISUALIZE**: The calculation to perform (`COUNT`, `AVG`, `P95`, `HEATMAP`, etc.)
- **WHERE**: Filters to narrow down results
- **GROUP BY**: How to segment the data
- **ORDER BY**: How to sort results

## Common Analysis Techniques

### 1. Finding Slow Requests

Identify which requests are taking the longest:

| Component | Value |
|-----------|-------|
| VISUALIZE | `HEATMAP(duration_ms)` |
| WHERE | `trace.parent_id does-not-exist` |
| GROUP BY | `name` or `http.route` |

Click on dots in the heatmap to drill into individual slow traces.

### 2. Identifying Bottlenecks

Find which agent nodes or MCP calls consume the most time:

| Component | Value |
|-----------|-------|
| VISUALIZE | `P95(duration_ms)` |
| WHERE | (none) |
| GROUP BY | `name` |

This reveals whether `agent.plan`, `agent.execute`, or specific MCP tools are the bottleneck.

### 3. Tracing Errors

Query for failed operations:

| Component | Value |
|-----------|-------|
| VISUALIZE | `COUNT` |
| WHERE | `error = true` or `mcp.success = false` |
| GROUP BY | `name`, `mcp.tool`, or `exception.message` |

### 4. MCP Tool Performance

Analyze which MCP servers are slowest:

| Component | Value |
|-----------|-------|
| VISUALIZE | `HEATMAP(mcp.duration_ms)` |
| WHERE | `name starts-with mcp.call_tool` |
| GROUP BY | `mcp.tool` |

### 5. LLM Call Latency

Track OpenAI API performance:

| Component | Value |
|-----------|-------|
| VISUALIZE | `AVG(duration_ms)` |
| WHERE | `name contains openai` |
| GROUP BY | `name` |

## Using BubbleUp for Root Cause Analysis

BubbleUp is Honeycomb's killer feature for finding *why* something is slow or failing.

### How to Use BubbleUp

1. Run a `HEATMAP(duration_ms)` query
2. Click and drag to select a region of interest (e.g., a cluster of slow requests)
3. Click the **BubbleUp** tab below the chart
4. Honeycomb compares your selection against the baseline and shows which attributes differ

### What BubbleUp Reveals

- Orange bars = your selected (problematic) data
- Blue bars = baseline (normal) data
- Attributes are ranked by how much they differ between the two sets

For example, BubbleUp might reveal that:
- All slow requests have `mcp.tool = search_resources`
- Errors cluster around a specific `session_id`
- High latency correlates with `mcp.server = xdmod-data`

### BubbleUp Best Practices

- Select data that clearly separates from the baseline (outliers, not the middle of the cluster)
- Use narrower time ranges to reduce unrelated variation
- Compare "bad" periods against known "good" periods

## Useful Queries for ACCESS-CI Agent

### Overall Health Dashboard

| Query Purpose | VISUALIZE | WHERE | GROUP BY |
|---------------|-----------|-------|----------|
| Request latency | `P95(duration_ms)` | `trace.parent_id does-not-exist` | `http.route` |
| Error rate | `COUNT` | `error = true` | `name` |
| Requests per minute | `RATE_AVG(COUNT)` | `trace.parent_id does-not-exist` | - |

### MCP Server Analysis

| Query Purpose | VISUALIZE | WHERE | GROUP BY |
|---------------|-----------|-------|----------|
| Tool usage | `COUNT` | `name starts-with mcp.call_tool` | `mcp.tool` |
| Tool failures | `COUNT` | `mcp.success = false` | `mcp.tool`, `mcp.server` |
| Tool latency | `HEATMAP(duration_ms)` | `name starts-with mcp.call_tool` | `mcp.tool` |
| Slowest tools | `P99(duration_ms)` | `name starts-with mcp.call_tool` | `mcp.tool` |

### Agent Node Performance

| Query Purpose | VISUALIZE | WHERE | GROUP BY |
|---------------|-----------|-------|----------|
| Node timing | `AVG(duration_ms)` | `name starts-with agent.` | `name` |
| Classification distribution | `COUNT` | `name = agent.classify` | `agent.query_type` |
| Retry frequency | `COUNT` | `name = agent.evaluate` | `agent.quality_passed` |

### Session Analysis

| Query Purpose | VISUALIZE | WHERE | GROUP BY |
|---------------|-----------|-------|----------|
| Queries per session | `COUNT` | `trace.parent_id does-not-exist` | `session_id` |
| Session errors | `COUNT` | `error = true` | `session_id` |

## Debugging Workflows

### Investigating a Slow Request

1. Start with `HEATMAP(duration_ms)` on root spans
2. Click a slow dot to open the trace
3. In the waterfall, identify the longest span
4. Check span attributes for clues (tool arguments, response size, etc.)
5. Use BubbleUp to compare against faster requests

### Investigating Errors

1. Query `COUNT` where `error = true`, grouped by `exception.message`
2. Click a count to see example traces
3. In the trace, find the red error span
4. Check the span's `exception.stacktrace` attribute
5. Look at parent spans for context (what was the request?)

### Comparing Before/After a Deployment

1. Set time range to include both periods
2. Run `HEATMAP(duration_ms)` query
3. Select the "after" time period
4. Use BubbleUp to see what changed

## Pro Tips

### Filtering Techniques

- `does-not-exist` - Find root spans: `trace.parent_id does-not-exist`
- `starts-with` - Match span names: `name starts-with mcp.call_tool`
- `contains` - Partial matching: `name contains openai`
- `in` - Multiple values: `mcp.tool in search_resources, get_infrastructure_news`

### Saved Queries

Save frequently-used queries for quick access:
1. Run your query
2. Click "Save" in the query builder
3. Give it a descriptive name
4. Access from the Saved Queries menu

### Trace Links

Your traces include `session_id` - use this to follow a user's entire conversation:

| Component | Value |
|-----------|-------|
| WHERE | `session_id = "specific-session-id"` |
| ORDER BY | `timestamp` |

### Service Map

Check the **Service Map** view to visualize how the agent connects to MCP servers. This shows:
- Request flow between services
- Latency between hops
- Error rates on each connection

## References

- [Get Started with Traces | Honeycomb](https://docs.honeycomb.io/get-started/start-building/application/traces/)
- [Investigating Timeouts with Tracing | Honeycomb](https://www.honeycomb.io/blog/investigating-timeouts-with-tracing)
- [Examples: Query for Traces | Honeycomb](https://docs.honeycomb.io/investigate/query/examples-traces/)
- [Heatmaps + BubbleUp: How They Work | Honeycomb](https://www.honeycomb.io/resources/heatmaps-bubbleup-how-they-work)
- [Build a Query | Honeycomb](https://docs.honeycomb.io/investigate/query/build/)
- [Identify Outliers | Honeycomb](https://docs.honeycomb.io/investigate/analyze/identify-outliers/)
