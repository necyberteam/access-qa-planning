# ACCESS QA System Planning

Planning documentation for the ACCESS-CI intelligent documentation agent.

## What We're Building

An AI-powered question-answering system for [ACCESS](https://access-ci.org) (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support). ACCESS allocates computing resources from Resource Providers — supercomputers, cloud platforms, and storage systems — to researchers across the US.

Users ask questions like:
- "What GPUs does Delta have?"
- "How do I request an allocation?"
- "Is Expanse currently down?"
- "Open a support ticket about my login issue"

The agent classifies each query and routes it to the appropriate handler: documentation questions are answered from UKY's document RAG, real-time questions use live MCP tool calls, and combined queries run both in parallel. Domain agents handle specialized workflows (ticket creation, announcement management). All responses include citations to source data.

## System Architecture

```
                              ┌─────────────────┐
                              │  Chatbot UI     │
                              │  (qa-bot-core)  │
                              └────────┬────────┘
                                       │
                              ┌────────┴────────┐
                              │  access-agent   │  LangGraph orchestration
                              │  (Python)       │  Classify → Route → Synthesize
                              └──┬─────┬─────┬──┘
                                 │     │     │
                    ┌────────────┘     │     └────────────┐
                    ▼                  ▼                  ▼
           ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
           │  UKY RAG     │   │  MCP Servers │   │  Domain      │
           │  (document   │   │  (11 servers,│   │  Agents      │
           │   retrieval) │   │  TS + Python)│   │  (JSM,       │
           └──────────────┘   └──────┬───────┘   │  Announce.)  │
                                     │           └──────────────┘
                                     ▼
                              ┌──────────────┐
                              │  ACCESS APIs │
                              │  (live data) │
                              └──────────────┘

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  PostgreSQL  │   │  Eval        │   │  Argilla     │
  │  (usage logs,│   │  Pipeline    │──▶│  (human      │
  │  eval scores,│   │  (LLM judge) │◀──│   review)    │
  │  checkpoints)│   └──────────────┘   └──────────────┘
  └──────────────┘
```

## Reading Paths

**New to the project?** Start with [Agent Architecture](active/01-agent-architecture.md) for the full system design, then read the [Decisions](#decisions) to understand why key choices were made.

**Working on MCP tools or authentication?**
1. [MCP Action Tools](active/05-events-actions.md) — overview and patterns
2. [MCP Authentication](active/06-mcp-authentication.md) — OAuth 2.1, CILogon, token flow

**Working on the chatbot UI or capabilities?**
1. [Capability Registry](active/11-capability-registry.md) — dynamic UI, ratings
2. [Resource-Scoped Capabilities](active/resource-scoped-capabilities.md) — RP-specific embedding

**Working on evaluation or reporting?**
1. [Decision 005](decisions/005-llm-judge-eval-pipeline.md) — why we built the eval pipeline
2. Eval pipeline usage: `access-agent/eval/README.md`

## Decisions

Key architectural and design decisions. Read these to understand *why* the system works the way it does.

| # | Decision | Date | Summary |
|---|----------|------|---------|
| [001](decisions/001-rag-over-finetuning.md) | RAG over fine-tuning | 2025-12 | Fine-tuned models hallucinated; RAG retrieves verified answers |
| [002](decisions/002-uky-rag-over-pgvector.md) | UKY RAG over pgvector pairs | 2026-02 | UKY docs broader coverage; Q&A pairs too narrow for real queries |
| [003](decisions/003-uky-first-architecture.md) | UKY-first architecture | 2026-03 | Always consult UKY regardless of classification |
| [004](decisions/004-jsm-dry-run-safeguard.md) | JSM dry-run safeguard | 2026-03 | Test runs created real tickets; MCP server needs dry-run mode |
| [005](decisions/005-llm-judge-eval-pipeline.md) | LLM-as-judge eval pipeline | 2026-03 | 5-dimension rubric + Argilla for human calibration |
| [006](decisions/006-onprem-llm-for-eval.md) | On-premise LLM for eval | 2026-03 | Privacy policy doesn't cover external LLM processing of user queries |

## Active Documents

Current planning and design docs. These describe how things work now and what's planned.

| Document | Purpose |
|----------|---------|
| [Agent Architecture](active/01-agent-architecture.md) | System design, routing, synthesis |
| [Review System](active/03-review-system.md) | Argilla for Q&A curation and eval review |
| [MCP Action Tools](active/05-events-actions.md) | Announcements, events, JSM ticket creation |
| [MCP Authentication](active/06-mcp-authentication.md) | OAuth 2.1, CILogon, token strategy |
| [Observability](active/08-observability.md) | Honeycomb, OpenTelemetry, quality eval |
| [Researcher Profiles](active/09-researcher-profiles.md) | User personalization (future) |
| [Analytics & Domain Agents](active/10-analytics-and-domain-agents.md) | Weekly reports, GA4, domain agent routing |
| [Capability Registry](active/11-capability-registry.md) | Dynamic UI, ratings, personalized context |
| [Privacy Policy](active/privacy-policy-additions.md) | PII handling, LLM processing policy |
| [Resource-Scoped Capabilities](active/resource-scoped-capabilities.md) | RP-specific chatbot embedding |
| [Resource-Scoped RAG](active/uky-resource-scoped-rag-spec.md) | UKY endpoint integration per resource |

### Specs and Plans (in access-agent repo)

| Document | Purpose |
|----------|---------|
| [Capability Registry Spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-18-capability-registry-design.md) | Capability registry detailed spec |
| [Eval Pipeline Spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-31-eval-pipeline-design.md) | Eval pipeline design spec |
| [Eval README](https://github.com/necyberteam/access-agent/blob/main/eval/README.md) | Eval pipeline usage guide |

## Archive

Completed implementation specs and historical docs. These describe work that's done — the implementation is in the code.

| Document | Status |
|----------|--------|
| [Q&A Data Preparation](archive/02-qa-data.md) | pgvector approach on hold ([Decision 002](decisions/002-uky-rag-over-pgvector.md)) |
| [Model Training](archive/04-model-training.md) | Deprecated ([Decision 001](decisions/001-rag-over-finetuning.md)) |
| [Backend Integration Spec](archive/07-backend-integration-spec.md) | Implemented |
| [QA Bot Authentication](archive/08-qa-bot-authentication.md) | JWT cookie auth implemented |
| [Drupal Announcements API](archive/drupal-announcements-api-spec.md) | Implemented |
| [Drupal JWT Cookie Spec](archive/drupal-jwt-cookie-spec.md) | Implemented |
| [Honeycomb Trace Analysis](archive/honeycomb-trace-analysis.md) | Completed analysis |
| [JSM MCP Server Plan](archive/jsm-mcp-server-plan.md) | Implemented |
| [JSM My Tickets API](archive/jsm-my-tickets-api-spec.md) | Implemented |
| [MCP Extraction](archive/mcp-extraction-impl.md) | Implemented |
| [Access QA Tool Pages](archive/pages-access-qa-tool.md) | Historical mockups |
| [Current Production Pages](archive/pages-current-production.md) | Historical reference |
| [Turnstile Bot Protection](archive/turnstile-bot-protection-spec.md) | Implemented |

## Current Status

**Phase:** Production deploy in progress (April 2026)

**Deployed:**
- UKY-first RAG architecture with MCP tool calling ([Decision 003](decisions/003-uky-first-architecture.md))
- 11 MCP servers (compute resources, system status, software, XDMoD, allocations, NSF awards, events, announcements, affinity groups, JSM, xdmod-data)
- Domain agents for announcements CRUD and JSM ticket creation
- JWT cookie authentication (ES256 + JWKS)
- Weekly analytics reports (GA4 + PostgreSQL → email/Slack)
- Chatbot UI analytics (GTM → GA4 events)
- JSM dry-run safeguard ([Decision 004](decisions/004-jsm-dry-run-safeguard.md))
- Streamable HTTP transport (replacing SSE) for MCP servers
- CI pipeline (ruff, mypy, pytest on PRs via GitHub Actions)

**In Progress:**
- Capability registry — data models, classification, and ratings merged; API endpoints and UI integration in progress
- Agent eval pipeline — core pipeline built (LLM judge, Argilla integration, CLI), pre-deploy testing underway
- Production deploy of new agent to production server
- Resource-scoped RAG (UKY endpoint per resource)
- On-premise LLM at UKY for production eval scoring ([Decision 006](decisions/006-onprem-llm-for-eval.md))

**Next:**
- Turnstile bot protection deployment (code complete, needs Cloudflare keys)
- Deploy remaining ACCESS sites with JWT cookie support
- Eval pipeline production scoring (pending on-premise LLM)
- Interactive eval data explorer (web UI + LLM query)
- Agent self-eval capability (gated by Drupal roles)
- Judge calibration using human review data from Argilla

## Related Repositories

| Repository | Description |
|------------|-------------|
| [access-agent](https://github.com/necyberteam/access-agent) | LangGraph agent — classification, routing, synthesis, eval |
| [access-mcp](https://github.com/necyberteam/access_mcp) | MCP servers for ACCESS data (11 servers, TypeScript + Python) |
| [access-qa-service](https://github.com/necyberteam/access-qa-service) | Q&A retrieval service (pgvector, on hold) |
| [access-qa-extraction](https://github.com/necyberteam/access-qa-extraction) | Q&A pair extraction + Argilla push |
| [access-argilla](https://github.com/necyberteam/access-argilla) | Argilla deployment (Docker Compose) |
| [access-serverless-api-1](https://github.com/necyberteam/access-serverless-api-1) | Netlify serverless proxy for JSM |
| [access-qa-bot](https://github.com/necyberteam/access-qa-bot) | ACCESS-specific chatbot wrapper (analytics, auth) |
| [qa-bot-core](https://github.com/necyberteam/qa-bot-core) | Core chatbot UI component (react-chatbotify) |
