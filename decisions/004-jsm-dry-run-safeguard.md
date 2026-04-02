# Decision 004: JSM Dry-Run Safeguard

**Date:** 2026-03-27
**Status:** Accepted
**Decided by:** Drew, after incident on 2026-03-26

## Context

On 2026-03-26, six duplicate JSM tickets were created in the Delta and Bridges-2 queues during development testing. The MCP JSM server defaulted its proxy URL to the live Netlify function (`https://access-jsm-api.netlify.app`) when `JSM_PROXY_URL` was unset. Combined with the agent's retry loop (3 retries before the circuit breaker was added), this created real tickets in production Jira.

## Decision

Add a `JSM_DRY_RUN` mode to the MCP JSM server and remove the dangerous default proxy URL.

## Key Details

- `JSM_DRY_RUN=true|1|yes` short-circuits ticket creation, returns fake `DRYRUN-NNN` responses
- `JSM_PROXY_URL` no longer defaults to production — must be set explicitly
- Local docker-compose defaults to `JSM_DRY_RUN=true`
- Production docker-compose sets `JSM_DRY_RUN=false` and requires `JSM_PROXY_URL`

## Alternatives Considered

- **Agent-level blocking** — Would prevent testing the full domain agent flow
- **Netlify-level test flag** — Risk of accidentally blocking real user submissions

## See Also

- Spec: `access_mcp/docs/superpowers/specs/2026-03-27-jsm-dry-run-safeguard-design.md`
