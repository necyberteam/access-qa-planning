# Turnstile Bot Protection for Chatbot

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [Capability Registry](./11-capability-registry.md)

## Problem

The chatbot currently requires login to use, largely because the UKY RAG pipeline requires authentication. We want to open the chatbot to anonymous users while protecting the inference endpoints from bot abuse.

## Approach

Cloudflare Turnstile challenges verify the user is human. The agent controls the policy — the frontend just responds. Authenticated users (JWT cookie) skip Turnstile entirely.

## Configuration

Environment variables on the agent:

```
TURNSTILE_SITE_KEY=...          # Cloudflare site key (passed to frontend)
TURNSTILE_SECRET_KEY=...        # Cloudflare secret key (for server-side verification)
TURNSTILE_MODE=deferred         # "immediate" or "deferred"
TURNSTILE_FREE_QUERIES=3        # Queries before challenge (deferred mode only)
TURNSTILE_SESSION_TTL=3600      # Seconds a verified session lasts (default: 1 hour)
```

## Flow

### Anonymous user, deferred mode (default)

1. User opens chatbot, asks a question
2. Agent checks session — no JWT cookie, no Turnstile verification yet, free queries remaining
3. Agent processes the query normally, decrements free query count for this session
4. After N free queries, user asks another question
5. Agent returns `{"requires_turnstile": true, "site_key": "..."}` instead of an answer
6. Frontend renders the Turnstile widget inline in the chat
7. User completes the challenge (usually invisible/automatic)
8. Frontend resends the original query with `turnstile_token` in the request body
9. Agent verifies the token with Cloudflare's `/siteverify` endpoint
10. On success, marks the session as verified and processes the query
11. Subsequent queries from this session are processed without challenge for the TTL period

### Anonymous user, immediate mode

Same as above but step 2 triggers the challenge on the first query (free query count is 0).

### Authenticated user

JWT cookie is present and valid — skip all Turnstile checks. The user has already authenticated via CILogon.

### Verification failure

If the Turnstile token is invalid or expired, return `{"requires_turnstile": true, "site_key": "..."}` again so the frontend can retry. Don't reveal why it failed.

## Agent Changes (access-agent)

### Query endpoint (`src/api/routes.py`)

Before processing a query, check:

```python
# Skip for authenticated users
if acting_user:
    # process normally
    pass
elif requires_turnstile_check(session_id):
    token = request.turnstile_token
    if token and verify_turnstile(token):
        mark_session_verified(session_id)
        # process normally
    else:
        return {"requires_turnstile": True, "site_key": settings.TURNSTILE_SITE_KEY}
```

### Session tracking

Track per-session state in Redis (or in-memory dict for single instance):
- `query_count`: number of queries this session
- `verified`: whether Turnstile has been completed
- `verified_at`: timestamp of verification

Key by session ID (already sent with every query as `X-Session-ID`).

### Turnstile verification (`src/auth.py` or new `src/turnstile.py`)

```python
async def verify_turnstile(token: str) -> bool:
    response = await httpx.post(
        "https://challenges.cloudflare.com/turnstile/v0/siteverify",
        data={
            "secret": settings.TURNSTILE_SECRET_KEY,
            "response": token,
        },
    )
    return response.json().get("success", False)
```

### New config settings

```python
TURNSTILE_SITE_KEY: str = ""
TURNSTILE_SECRET_KEY: str = ""
TURNSTILE_MODE: Literal["immediate", "deferred"] = "deferred"
TURNSTILE_FREE_QUERIES: int = 3
TURNSTILE_SESSION_TTL: int = 3600
```

When `TURNSTILE_SECRET_KEY` is empty, Turnstile is disabled (current behavior — all anonymous queries blocked by login gate).

## Frontend Changes (qa-bot-core)

### Handle `requires_turnstile` response

When the agent returns `{"requires_turnstile": true, "site_key": "..."}`:

1. Render the Turnstile widget in the chat area (below the user's message)
2. On Turnstile success, get the token
3. Resend the original query with `turnstile_token` in the POST body
4. Show the agent's response as normal

The Turnstile widget can use Cloudflare's managed mode (usually invisible, only shows a challenge if suspicious).

### Turnstile script loading

Load the Turnstile script lazily — only when `requires_turnstile` is first received:

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
```

### Query request model

Add optional `turnstile_token` to the POST body:

```json
{
  "query": "...",
  "session_id": "...",
  "turnstile_token": "optional-token-here"
}
```

## Trust Levels

The agent tracks not just who the user is but how they were identified:

| Identity Source | Turnstile | General Queries | Authorized Actions |
|----------------|-----------|-----------------|-------------------|
| JWT cookie (signed server-side) | Skip | Yes | Yes |
| Body fallback (from embedding) | Skip | Yes | No |
| Anonymous (no identity) | Required | After verification | No |

**JWT cookie** is the only path that enables authorized actions (creating announcements, managing content, opening tickets as a user). The cookie is signed server-side so it can't be forged.

**Body fallback** is trusted for identifying the user (skip Turnstile, personalized queries) but not for acting as the user. Sites that pass `acting_user` via the embedding can use the chatbot fully for read operations, but write operations require the JWT cookie.

**Anonymous** users must pass Turnstile to use the chatbot at all. They get anonymous capabilities only (support tickets, general info if UKY supports it).

The agent already distinguishes these — `get_acting_user_from_cookie` returns `(user, cookie_present)`. When `cookie_present` is false and the user came from the body, the agent knows it's the fallback path and restricts accordingly.

## What Does NOT Change

- The login gate in qa-bot-core stays as a fallback — if Turnstile is not configured, behavior is the same as today
- UKY RAG supports anonymous access. They handle their own rate limiting and rely on us for bot protection (Turnstile).

## Open Questions

1. **Rate limiting after verification** — should verified anonymous sessions still have a per-hour query limit on our side? Turnstile proves human, but a human could still hammer the endpoint. UKY does their own rate limiting.
2. **Turnstile keys** — reuse the existing Drupal Turnstile site key, or create a separate one for the chatbot?
