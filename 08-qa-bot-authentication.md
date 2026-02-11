# QA Bot Authentication Architecture

## Overview

This document describes the authentication architecture for the QA Bot, enabling secure user identity attribution across multiple websites within an organization. While written for ACCESS-CI, this pattern is designed to be transferable to other programs and organizations.

### Design Goals

1. **Secure identity attribution** - Backend determines user identity from server-validated source
2. **Progressive enhancement** - Sites without authentication support fall back to anonymous mode
3. **Minimal site integration** - Sites only need to set a cookie; no backend proxying required
4. **Transferable pattern** - Architecture can be adopted by other programs beyond ACCESS-CI

## Problem Statement

The QA Bot component (`qa-bot-core`) is embedded across multiple ACCESS-CI websites. The backend needs to know which authenticated user is making requests (the "ActingUser") to:

1. Provide personalized responses based on user context
2. Enforce authorization for sensitive queries
3. Audit and attribute actions to specific users
4. Prevent spoofing of user identity

## Authentication vs. Abuse Protection

A key architectural decision is whether to **require authentication** to use the QA Bot, or allow anonymous access with other abuse protections.

### Why Require Authentication?

| Concern | Auth Required? | Alternatives |
|---------|---------------|--------------|
| **Cost control** | No | Rate limiting by IP, session, or API key |
| **Rate limiting** | No | IP-based, session-based, or fingerprinting |
| **Accountability** | Partially | Session IDs provide some traceability |
| **User-specific data** | **Yes** | No alternative - must know who's asking |

**Key insight:** Only user-specific data access truly requires authentication. Cost control and abuse prevention can be handled via rate limiting.

### Industry Patterns

| Service | Auth Required? | Abuse Protection |
|---------|---------------|------------------|
| ChatGPT (free tier) | Yes (account) | Account creation barrier |
| Perplexity | No | Rate limits, CAPTCHA |
| Google Search AI | No | Rate limits by IP/session |
| Intercom/Drift bots | No | Rate limits, session tracking |
| Most support chatbots | No | Session-based rate limits |

### Recommended Approach: Tiered Access

Rather than requiring authentication for all usage, implement tiered access:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TIERED ACCESS MODEL                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ANONYMOUS USERS                                                    │
│  └─→ General documentation Q&A                                     │
│  └─→ Public information queries                                    │
│  └─→ Rate limited (20 questions/hour, 50/day)                      │
│  └─→ No MCP tool calls requiring identity                          │
│  └─→ No user-specific data (allocations, tickets)                  │
│  └─→ Session-based tracking for rate limits and telemetry          │
│                                                                     │
│  AUTHENTICATED USERS                                                │
│  └─→ All anonymous features, plus:                                 │
│  └─→ Higher rate limits (100 questions/hour, no daily limit)       │
│  └─→ Personalized responses                                        │
│  └─→ User-specific data access (allocations, tickets, etc.)        │
│  └─→ MCP tool calls with ActingUser attribution                    │
│  └─→ Full audit trail with user identity                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Benefits of this approach:**

1. **Lower friction** - Casual visitors can get help without logging in
2. **Broader reach** - Reduces support burden by helping more people
3. **Data-driven** - Gather real usage data before adding restrictions
4. **Progressive** - Users naturally upgrade to auth for advanced features
5. **Reversible** - Can tighten restrictions if abuse becomes real

### Rate Limiting Strategy

**Anonymous users (by session ID + IP address):**

| Limit | Value | Rationale |
|-------|-------|-----------|
| Per hour | 20 questions | Prevents rapid abuse |
| Per day | 50 questions | Limits sustained abuse |
| Soft prompt | After 10 questions | "Login for unlimited access" |

**Authenticated users:**

| Limit | Value | Rationale |
|-------|-------|-----------|
| Per hour | 100 questions | Generous for legitimate use |
| Per day | No limit | Trust authenticated users |

**Implementation notes:**

- Rate limits keyed on `session_id + IP` for anonymous, `user_id` for authenticated
- Return `429 Too Many Requests` when limit exceeded
- Include `Retry-After` header with reset time
- Consider exponential backoff for repeated limit hits

### Soft Gate UX

To encourage authentication without requiring it:

```
┌─────────────────────────────────────────────────────────────────────┐
│  After 10 anonymous questions, show prompt:                        │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  💡 Login for unlimited questions and personalized answers    │ │
│  │                                                                │ │
│  │  With an ACCESS account, you can:                             │ │
│  │  • Ask unlimited questions                                    │ │
│  │  • Get answers about YOUR allocations and tickets            │ │
│  │  • Receive personalized recommendations                       │ │
│  │                                                                │ │
│  │  [Login with ACCESS] [Maybe later]                            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  User can dismiss and continue with anonymous access               │
└─────────────────────────────────────────────────────────────────────┘
```

### Telemetry for Future Decisions

To inform whether to adjust this approach, track:

| Metric | Purpose |
|--------|---------|
| Questions per session (anon vs auth) | Understand usage patterns |
| Drop-off at login prompts | Measure friction cost |
| Rate limit hits | Detect abuse, tune limits |
| Questions per unique IP | Identify abuse patterns |
| Cost per query | Budget impact of open access |
| Query categories | How many need user context |
| Conversion rate (anon → auth) | Soft gate effectiveness |

### Decision Triggers

Revisit this approach if:

| Trigger | Action to Consider |
|---------|-------------------|
| Abuse > 5% of queries | Add CAPTCHA for anonymous |
| Cost per query > $0.50 | Tighten anonymous limits |
| Rate limit hits > 10% of sessions | Limits may be too tight |
| Auth conversion < 1% | Soft gate not working |
| User-specific features launch | Promote auth benefits more |

### Current Status

- **User-specific features:** Not yet implemented (planned)
- **Abuse incidents:** None (theoretical risk only)
- **Cost per query:** Not yet measured
- **Friction data:** Telemetry being implemented

**Recommendation:** Start with tiered access (anonymous allowed), gather data, adjust based on real-world usage patterns.

### Security Challenge

Since the QA Bot is a frontend component, we cannot trust user identity passed from the client. A malicious actor could:

- Call the backend API directly with a spoofed `X-Acting-User` header
- Modify frontend code to claim a different identity
- Intercept and replay requests with altered identity

**Solution:** The backend must determine user identity from a server-validated source, not from client-supplied data.

## Architecture

### Shared Cookie Authentication

All ACCESS-CI services operate under the `*.access-ci.org` domain, enabling shared cookie authentication.

### System Components

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  QA Bot         │     │  QA Bot         │     │  MCP            │     │  Backend        │
│  Frontend       │────▶│  Agent          │────▶│  Server         │────▶│  Services       │
│  (qa-bot-core)  │     │  (LLM + RAG)    │     │  (tools)        │     │  (Drupal, etc.) │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
     Browser             qa.access-ci.org        mcp.access-ci.org       Various domains
```

| Component | Description | Auth Responsibility |
|-----------|-------------|---------------------|
| **QA Bot Frontend** | React component (`qa-bot-core`) embedded on ACCESS sites | Sends cookie via `credentials: 'include'` |
| **QA Bot Agent** | API service that receives questions, calls LLM, performs RAG | Validates cookie, extracts ActingUser, enforces rate limits |
| **MCP Server** | Provides tools for user-specific data (allocations, tickets) | Trusts `X-Acting-User` header from Agent (service-to-service auth) |
| **Backend Services** | Drupal, JSM, Allocations API, etc. | Trusts `X-Acting-User` header from MCP |

### Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. User logs into any ACCESS site (e.g., allocations.access-ci.org)│
│     └─→ CILogon authentication                                     │
│     └─→ Site sets session cookie: domain=.access-ci.org            │
│                                                                     │
│  2. User visits different ACCESS site with QA Bot embedded          │
│     └─→ Session cookie is present (same parent domain)             │
│                                                                     │
│  3. QA Bot Frontend sends question to QA Bot Agent                  │
│     └─→ Browser automatically includes cookie                      │
│     └─→ credentials: 'include' in fetch request                    │
│                                                                     │
│  4. QA Bot Agent validates session cookie                           │
│     └─→ Verifies JWT signature                                     │
│     └─→ Extracts ACCESS ID from token claims                       │
│     └─→ ActingUser determined server-side (cannot be spoofed)      │
│                                                                     │
│  5. QA Bot Agent calls MCP Server (if needed for user-specific data)│
│     └─→ Service-to-service auth (API key or JWT)                   │
│     └─→ Passes X-Acting-User header with validated ACCESS ID       │
│                                                                     │
│  6. MCP Server calls Backend Services                               │
│     └─→ Service-to-service auth                                    │
│     └─→ Passes X-Acting-User header                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Why This Prevents Spoofing

1. **Cookie is httpOnly** - JavaScript cannot read or forge it
2. **Cookie is signed** - User cannot modify the ACCESS ID without invalidating the signature
3. **Server-side resolution** - Client never sends ActingUser; backend extracts it from validated cookie
4. **Domain-scoped** - Cookie only sent to `*.access-ci.org` domains

### Graceful Degradation

Not all sites may support authenticated cookies immediately. The system supports progressive rollout with two authentication levels:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION LEVELS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Level 0: Anonymous                                                 │
│  └─→ No session cookie, or cookie invalid/expired                  │
│  └─→ QA Bot works, but no personalization or user-specific features│
│  └─→ Responses limited to public information                       │
│                                                                     │
│  Level 1: Authenticated                                             │
│  └─→ Valid session cookie with user identity                       │
│  └─→ Full personalization and user-specific features               │
│  └─→ Can access user's allocations, tickets, etc.                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Backend behavior by authentication level:**

| Feature | Anonymous (Level 0) | Authenticated (Level 1) |
|---------|---------------------|-------------------------|
| General Q&A | ✓ Enabled | ✓ Enabled |
| Public documentation queries | ✓ Enabled | ✓ Enabled |
| Personalized responses | ✗ Disabled | ✓ Enabled |
| User-specific data (allocations, tickets) | ✗ Disabled | ✓ Enabled |
| MCP tool calls requiring identity | ✗ Disabled | ✓ Enabled |
| Audit trail with user attribution | Anonymous session only | Full user attribution |

This allows sites to adopt the QA Bot immediately and add authentication support later.

### Frontend/Agent Auth Coordination

The QA Bot uses a hybrid approach where the frontend controls UI gating and the Agent determines actual authentication level:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HYBRID AUTH MODEL                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FRONTEND (QA Bot Component)                                        │
│  └─→ Parent app sets `isLoggedIn` prop based on its own auth state │
│  └─→ Typically: isLoggedIn = document.cookie.includes('session')   │
│  └─→ Controls whether to show login gate or allow questions        │
│  └─→ Does NOT guarantee authenticated responses                    │
│                                                                     │
│  QA BOT AGENT                                                       │
│  └─→ Validates session cookie on every request                     │
│  └─→ Cookie valid? → Authenticated response with personalization   │
│  └─→ Cookie invalid/missing? → Anonymous response (NOT an error)   │
│  └─→ Never returns 401 for missing auth - gracefully degrades      │
│  └─→ Returns auth status in response for frontend awareness        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Why this approach:**

1. **No extra network request** - Frontend doesn't need to call an auth-check endpoint on load
2. **No jarring errors** - If cookie expires mid-session, user still gets anonymous responses
3. **Progressive enhancement** - Sites can enable the bot before implementing JWT cookies
4. **Agent is authoritative** - Actual auth level determined by validated cookie, not client claim

**Response format includes auth status:**

```json
{
  "response": "Here's information about allocations...",
  "auth": {
    "authenticated": true,
    "user_id": "jsmith@access-ci.org"
  }
}
```

This allows the frontend to optionally:
- Display "Logged in as jsmith@access-ci.org"
- Show "Login for personalized answers" if `authenticated: false`
- Update UI if auth state changes mid-session

**Frontend `isLoggedIn` prop behavior:**

| `isLoggedIn` Prop | Cookie State | UI Behavior | Backend Behavior |
|-------------------|--------------|-------------|------------------|
| `false` | Any | Show login gate | N/A (no requests made) |
| `true` | Valid JWT | Allow questions | Authenticated responses |
| `true` | Invalid/expired JWT | Allow questions | Anonymous responses |
| `true` | No cookie | Allow questions | Anonymous responses |
| `true` | Old `1` value | Allow questions | Anonymous responses |

The frontend trusts the parent app's `isLoggedIn` prop for UI purposes, but the backend independently validates the cookie for actual authentication.

## Cookie Specification

### Cookie Structure

The session cookie contains a signed JWT with the user's ACCESS ID:

```
accessci_session=<signed-jwt>
```

### JWT Payload

```json
{
  "sub": "jsmith@access-ci.org",
  "iat": 1707000000,
  "exp": 1707086400
}
```

| Claim | Description |
|-------|-------------|
| `sub` | ACCESS ID (from CILogon eppn claim) |
| `iat` | Issued at timestamp (Unix epoch) |
| `exp` | Expiration timestamp (recommended: 24 hours after issuance) |

### JWT Signature

- **Algorithm:** HS256 (HMAC-SHA256)
- **Secret:** Shared secret distributed to all ACCESS sites and QA Bot Agent
- **Validation:** Backend MUST verify signature before trusting claims

### Cookie Attributes

```
Set-Cookie: accessci_session=<jwt>;
  Domain=.access-ci.org;
  Path=/;
  HttpOnly;
  Secure;
  SameSite=None
```

| Attribute | Value | Rationale |
|-----------|-------|-----------|
| `Domain` | `.access-ci.org` | Shared across all ACCESS subdomains |
| `Path` | `/` | Available to all paths |
| `HttpOnly` | `true` | Prevents JavaScript access (XSS protection) |
| `Secure` | `true` | HTTPS only |
| `SameSite` | `None` | Required for cross-subdomain AJAX requests |

### SameSite Explanation

`SameSite=None` is required because:

| Request Type | SameSite=Lax | SameSite=None |
|--------------|--------------|---------------|
| Top-level navigation | ✓ Sent | ✓ Sent |
| Same-subdomain AJAX | ✓ Sent | ✓ Sent |
| **Cross-subdomain AJAX** | ✗ Not sent | ✓ Sent |

When QA Bot on `allocations.access-ci.org` makes AJAX requests to `qa.access-ci.org`, the cookie must be sent. This requires `SameSite=None`.

**Security note:** `SameSite=None` is safe in this context because:
- `Secure` ensures HTTPS-only transmission
- `HttpOnly` prevents JavaScript access
- `Domain=.access-ci.org` limits cookie scope to ACCESS domains
- JWT signature prevents tampering

## Implementation Requirements

### ACCESS Sites (Cookie Issuers)

Each ACCESS site that authenticates users must:

1. **After CILogon authentication**, create a signed JWT:

```python
import jwt
import time

def create_session_cookie(access_id: str, secret: str) -> str:
    payload = {
        "sub": access_id,  # e.g., "jsmith@access-ci.org"
        "iat": int(time.time()),
        "exp": int(time.time()) + 86400  # 24 hours
    }
    return jwt.encode(payload, secret, algorithm="HS256")
```

2. **Set the cookie** with proper attributes:

```python
response.set_cookie(
    "accessci_session",
    value=create_session_cookie(user.access_id, SHARED_SECRET),
    domain=".access-ci.org",
    path="/",
    httponly=True,
    secure=True,
    samesite="None"
)
```

3. **Store the shared signing secret** securely (environment variable, secrets manager)

### QA Bot Frontend (`qa-bot-core`)

The frontend must include credentials with all API requests:

```typescript
// src/utils/flows/qa-flow.tsx

const response = await fetch(qaEndpoint, {
  method: 'POST',
  credentials: 'include',  // Sends cookies cross-origin
  headers: {
    'Content-Type': 'application/json',
    'X-Session-ID': sessionId,
    'X-Query-ID': queryId
  },
  body: JSON.stringify({ query })
});
```

**Important:** Remove any client-side `X-Acting-User` header. The backend determines identity from the cookie.

### QA Bot Agent

The backend must:

1. **Be deployed to `*.access-ci.org`** (e.g., `qa.access-ci.org`) to receive the cookie

2. **Configure CORS** to allow credentialed requests from ACCESS sites only:

```python
from flask_cors import CORS

CORS(app,
     origins=["https://*.access-ci.org"],
     supports_credentials=True)
```

**This is a hard security requirement**, not optional configuration. CORS is the primary defense against browser-based attacks from spoofed hosts:

| Scenario | CORS Behavior |
|----------|--------------|
| Request from `support.access-ci.org` | Allowed — origin matches `*.access-ci.org` |
| Request from `evil.com` with chatbot clone | Blocked — origin doesn't match, browser won't send cookie |
| Request from `evil.access-ci.org` | Allowed — matches wildcard. Only a risk if attacker controls an ACCESS subdomain |
| Server-side curl (no browser) | Not blocked by CORS — this is what JWT cookie validation prevents |

The agent should also validate the `Origin` header server-side as defense-in-depth (browsers send this honestly for cross-origin requests).

3. **Validate the session cookie** on every request (with graceful degradation):

```python
import jwt

SHARED_SECRET = os.environ["SESSION_SECRET"]

def get_acting_user(request) -> str | None:
    """Extract and validate ActingUser from session cookie.

    Returns None if cookie is missing, invalid, or expired.
    This is NOT an error condition - it means anonymous mode.
    """
    cookie = request.cookies.get("accessci_session")
    if not cookie:
        return None

    # Handle legacy cookie value (just "1" for logged-in flag)
    if cookie == "1":
        return None

    try:
        payload = jwt.decode(
            cookie,
            SHARED_SECRET,
            algorithms=["HS256"]
        )
        return payload["sub"]  # ACCESS ID
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None


@app.post("/qa")
def handle_qa():
    acting_user = get_acting_user(request)

    # Process query - works for both authenticated and anonymous users
    result = process_query(
        query=request.json["query"],
        acting_user=acting_user  # None for anonymous
    )

    # Include auth status in response so frontend can update UI
    return jsonify({
        "response": result.response,
        "auth": {
            "authenticated": acting_user is not None,
            "user_id": acting_user  # None if anonymous
        }
    })
```

4. **Pass ActingUser to downstream services** (MCP, etc.) via header:

```python
def call_mcp_tool(tool_name: str, params: dict, acting_user: str):
    response = requests.post(
        f"{MCP_ENDPOINT}/tools/{tool_name}",
        headers={
            "Authorization": f"Bearer {MCP_SERVICE_TOKEN}",
            "X-Acting-User": acting_user,  # Now trusted - came from validated cookie
            "X-Request-ID": str(uuid.uuid4())
        },
        json=params
    )
    return response.json()
```

## Security Considerations

### Shared Secret Management

All cookie issuers (ACCESS sites) and validators (QA Bot Agent) share a **single** JWT signing secret. This is what makes the cookie portable across `*.access-ci.org` — a cookie issued by `allocations.access-ci.org` can be validated by `qa.access-ci.org` because they share the same secret.

The signing secret must be:

- **Sufficiently random:** Minimum 256 bits of entropy
- **Securely stored:** In a secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.), not in code repositories or config files
- **Automatically distributed:** Services fetch from the secrets manager, not manually configured

#### Who Needs the Secret

| Service | Role | Needs Secret |
|---------|------|-------------|
| ACCESS Drupal sites | Issue cookies | Yes — signs JWTs |
| Other authenticated ACCESS sites | Issue cookies | Yes — signs JWTs |
| QA Bot Agent (`qa.access-ci.org`) | Validate cookies | Yes — verifies JWT signatures |
| MCP servers | N/A — trusts agent via service auth | No |
| JSM proxy | N/A — trusts agent via API key | No |
| Dumb hosts (WordPress, static) | N/A — just embeds chatbot widget | No |

#### Secret Rotation

Secret rotation must be automated and coordinated across all services. The 24-hour cookie lifetime provides a natural rotation window.

**Automated rotation process:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECRET ROTATION TIMELINE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  T+0:    Secrets manager generates new secret                       │
│          Both old and new secrets are marked "active"               │
│                                                                     │
│  T+0 to T+1h:  Services pick up new secret                         │
│          - Poll-based: services check secrets manager periodically  │
│          - Or push-based: webhook/notification triggers restart     │
│          - Services now accept JWTs signed with EITHER secret       │
│          - New cookies issued with NEW secret only                  │
│                                                                     │
│  T+1h to T+24h: Transition window                                  │
│          - Old cookies (signed with old secret) still valid         │
│          - New cookies (signed with new secret) being issued        │
│          - Both secrets accepted for validation                     │
│                                                                     │
│  T+24h:  All old cookies have expired                               │
│          Old secret retired from secrets manager                    │
│          Services stop accepting old secret on next poll            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation for multi-secret validation:**

```python
import jwt
import os
import json

def get_signing_secrets() -> list[str]:
    """Fetch active secrets from secrets manager.
    Returns list of secrets - first is current (for signing),
    rest are previous (for validation during rotation).
    """
    # Example: secrets manager returns JSON array
    secrets = json.loads(os.environ["SESSION_SECRETS"])
    return secrets  # ["new_secret", "old_secret"]

def create_session_cookie(access_id: str) -> str:
    """Sign with current (first) secret only."""
    secrets = get_signing_secrets()
    payload = {
        "sub": access_id,
        "iat": int(time.time()),
        "exp": int(time.time()) + 86400
    }
    return jwt.encode(payload, secrets[0], algorithm="HS256")

def get_acting_user(request) -> str | None:
    """Validate against all active secrets."""
    cookie = request.cookies.get("accessci_session")
    if not cookie or cookie == "1":
        return None

    for secret in get_signing_secrets():
        try:
            payload = jwt.decode(cookie, secret, algorithms=["HS256"])
            return payload["sub"]
        except jwt.InvalidTokenError:
            continue

    return None  # No secret could validate — anonymous mode
```

#### Recommendation: HashiCorp Vault

**Why Vault:**

- Open source, runs as a Docker container alongside the existing stack
- Native secret versioning — stores current and previous secret versions, exactly what multi-secret rotation needs
- API-driven — services fetch secrets via HTTP, no SDK lock-in
- Runs on any infrastructure (Linode, cloud, on-prem) — no vendor lock-in
- Industry standard for secret management

**Why not Kubernetes:** The current infrastructure is a single Linode running Docker Compose with ~10 services. K8s would add significant operational overhead (multi-node clusters, Helm charts, ingress controllers, K8s knowledge requirements) without meaningful benefit at this scale. Docker Compose with `restart: always` and health checks provides sufficient orchestration. Revisit if the system grows to multiple servers or requires auto-scaling.

**Docker Compose addition:**

```yaml
vault:
  image: hashicorp/vault:1.15
  cap_add:
    - IPC_LOCK
  environment:
    - VAULT_ADDR=http://0.0.0.0:8200
  volumes:
    - vault-data:/vault/data
    - ./vault/config:/vault/config
  ports:
    - "8200:8200"
  command: server
  restart: always
```

**How services consume the secret:**

```python
import hvac  # Vault Python client

vault = hvac.Client(url="http://vault:8200", token=os.environ["VAULT_TOKEN"])

def get_signing_secrets() -> list[str]:
    """Fetch current + previous secret versions from Vault."""
    secret = vault.secrets.kv.v2.read_secret_version(
        path="accessci/session-signing",
        mount_point="secret"
    )
    current = secret["data"]["data"]["key"]

    # Also fetch previous version for rotation window
    try:
        previous = vault.secrets.kv.v2.read_secret_version(
            path="accessci/session-signing",
            version=secret["data"]["metadata"]["version"] - 1,
            mount_point="secret"
        )
        return [current, previous["data"]["data"]["key"]]
    except Exception:
        return [current]
```

**Rotation automation:** A cron job (or Vault's own rotation policies) generates a new secret on schedule (e.g., every 30 days). Writing a new version to Vault's KV v2 engine automatically preserves the previous version. Services polling Vault pick up both versions and accept JWTs signed with either. After 24 hours, all cookies signed with the old secret have expired.

**Distributing to ACCESS sites (Drupal):** The Drupal sites that issue cookies also need the current signing secret. Options:
- Drupal module that polls Vault on a cron schedule
- Vault Agent running alongside Drupal, writing the secret to a file that Drupal reads
- For sites that can't run Vault Agent, a simple API endpoint on the agent that returns the current signing key (authenticated with a separate service token)

#### Vault Availability & Fallback

Services must handle Vault being temporarily unreachable:

| Scenario | Behavior |
|----------|----------|
| Vault reachable | Fetch and cache secrets with TTL (5-10 minutes) |
| Vault unreachable, cache valid | Use cached secrets — continue normally |
| Vault unreachable, cache expired | **Fail closed** — reject authenticated operations, allow anonymous Q&A |
| Vault unreachable on startup | Fail to start — require manual intervention |

```python
import time

_secret_cache = {"secrets": None, "fetched_at": 0}
CACHE_TTL = 300  # 5 minutes

def get_signing_secrets() -> list[str]:
    now = time.time()
    if _secret_cache["secrets"] and (now - _secret_cache["fetched_at"]) < CACHE_TTL:
        return _secret_cache["secrets"]

    try:
        secrets = fetch_from_vault()
        _secret_cache["secrets"] = secrets
        _secret_cache["fetched_at"] = now
        return secrets
    except VaultUnavailableError:
        if _secret_cache["secrets"]:
            # Vault down but cache still warm — use cached secrets
            return _secret_cache["secrets"]
        # No cache, no Vault — fail closed
        raise
```

**Key principle:** Fail closed for authentication (don't guess), fail open for anonymous Q&A (don't break the chatbot for unauthenticated users just because Vault is down).

### Token Expiration

- **Recommended lifetime:** 24 hours
- **Behavior on expiration:** User must re-authenticate on any ACCESS site
- **No refresh mechanism:** Cookie is re-issued on each site login

### Cookie Expiry Mid-Session

When a JWT cookie expires during an active chatbot session:

| Component | Behavior |
|-----------|----------|
| **Agent** | Returns anonymous response (not an error). Includes `auth.authenticated: false` in response. |
| **Chatbot frontend** | Detects `authenticated: false` in response. If user was previously authenticated, shows a non-blocking prompt: "Your session has expired. [Log in again] to access personalized features." |
| **Q&A** | Continues working — anonymous mode, no interruption |
| **Ticket operations** | Blocked — agent returns "Please log in to access ticket features" instead of proceeding with unverified identity |

The user's chat history and session are preserved. Only identity-dependent operations are affected. Re-authenticating on any ACCESS site restores the cookie immediately.

### Logout Considerations

When a user logs out:

1. **Clear the cookie** on the site they logged out from
2. **Cookie persists on other subdomains** until expiration (this is expected SSO behavior)
3. **For immediate logout everywhere:** Consider a token revocation list (adds complexity)

### Attack Vectors Mitigated

| Attack | Mitigation |
|--------|------------|
| Cookie theft via XSS | `HttpOnly` flag prevents JavaScript access |
| Cookie theft via network sniffing | `Secure` flag ensures HTTPS only |
| Cookie tampering | JWT signature verification |
| Cross-site request forgery | `SameSite=None` with `Secure`; consider CSRF tokens for state-changing operations |
| Session fixation | Issue new cookie on authentication |
| Replay attacks | Token expiration limits window |

### CSRF Considerations

With `SameSite=None`, CSRF protection should be considered for state-changing operations. Options:

1. **CSRF tokens** for write operations
2. **Check `Origin` header** matches expected ACCESS domains
3. **Require additional confirmation** for sensitive actions

For read-only QA queries, CSRF risk is lower, but write operations (if any) should have additional protection.

## Migration Path

### Current State

- Cookie value is `1` (boolean flag)
- Cookie has `SameSite=Lax`
- QA Bot Agent not on `*.access-ci.org`

### Steps to Implement

1. **Deploy QA Bot Agent to `qa.access-ci.org`**
   - Configure DNS
   - Set up SSL certificate
   - Configure CORS for credentialed requests

2. **Generate and distribute shared secret**
   - Generate cryptographically secure secret
   - Add to secrets manager
   - Distribute to all ACCESS sites and QA Bot Agent

3. **Update ACCESS sites to issue JWT cookies**
   - Modify authentication handlers
   - Change cookie value from `1` to signed JWT
   - Update `SameSite` from `Lax` to `None`

4. **Update QA Bot frontend**
   - Add `credentials: 'include'` to fetch calls
   - Remove any client-side ActingUser headers

5. **Update QA Bot Agent**
   - Add JWT validation logic
   - Extract ActingUser from validated token
   - Pass to downstream services

### Rollback Plan

If issues arise:

1. Sites can revert to `SameSite=Lax` and cookie value `1`
2. QA Bot Agent can fall back to anonymous mode
3. Changes are independent per-site, allowing gradual rollout

## Appendix: Example JWT

### Encoded

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqc21pdGhAYWNjZXNzLWNpLm9yZyIsImlhdCI6MTcwNzAwMDAwMCwiZXhwIjoxNzA3MDg2NDAwfQ.X5qR7mK9vJ2nL8pF3wY6hT1uD4cA0bE9iO2sN7gM5kQ
```

### Decoded Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Decoded Payload

```json
{
  "sub": "jsmith@access-ci.org",
  "iat": 1707000000,
  "exp": 1707086400
}
```

## Adapting for Other Programs

This authentication pattern is designed to be transferable to other programs beyond ACCESS-CI. The core requirements are:

1. **Shared parent domain** - All sites must be under a common domain (e.g., `*.myprogram.org`)
2. **Common identity provider** - A way to authenticate users and obtain a canonical user ID
3. **Shared signing secret** - Distributed to all sites and the QA Bot Agent

### Configuration Points

To adapt for another program, modify these values:

| Parameter | ACCESS-CI Value | Your Program |
|-----------|-----------------|--------------|
| Cookie name | `accessci_session` | `{program}_session` |
| Cookie domain | `.access-ci.org` | `.{your-domain}` |
| User ID claim | `sub` (ACCESS ID) | `sub` (your canonical ID) |
| Identity provider | CILogon | Your IdP (Okta, Auth0, SAML, etc.) |
| Backend domain | `qa.access-ci.org` | `qa.{your-domain}` |

### Minimal Implementation Checklist

For a new program to adopt this pattern:

- [ ] **Choose cookie name** - e.g., `myprogram_session`
- [ ] **Generate shared secret** - 256+ bits, store in secrets manager
- [ ] **Deploy QA Bot Agent** - On a subdomain of your shared domain
- [ ] **Configure backend** - Set cookie name, domain, and signing secret
- [ ] **Update sites to issue JWT cookies** - After user authentication
- [ ] **Set cookie attributes** - `HttpOnly`, `Secure`, `SameSite=None`, `Domain=.{your-domain}`

### Example: Adapting for "MyResearch" Program

```python
# Configuration for MyResearch program
COOKIE_NAME = "myresearch_session"
COOKIE_DOMAIN = ".myresearch.org"
SIGNING_SECRET = os.environ["MYRESEARCH_SESSION_SECRET"]

# JWT payload uses same structure
payload = {
    "sub": "researcher123@myresearch.org",  # Canonical user ID
    "iat": int(time.time()),
    "exp": int(time.time()) + 86400
}

# Cookie settings
response.set_cookie(
    COOKIE_NAME,
    value=jwt.encode(payload, SIGNING_SECRET, algorithm="HS256"),
    domain=COOKIE_DOMAIN,
    httponly=True,
    secure=True,
    samesite="None"
)
```

### Programs Without Shared Domain

If your program's sites are on different domains (e.g., `site-a.com`, `site-b.org`), the shared cookie approach won't work. Alternative options:

1. **OAuth flow** - QA Bot authenticates users directly via OAuth (see Option A/B in architecture discussion)
2. **Token prop** - Parent sites pass a signed token to the QA Bot component
3. **Subdomain consolidation** - Move sites to a shared domain if feasible

The shared cookie approach is simplest when a common domain exists.

## Related Documents

- [06-mcp-authentication.md](./06-mcp-authentication.md) - MCP OAuth architecture
- [07-backend-integration-spec.md](./07-backend-integration-spec.md) - Backend API patterns including X-Acting-User header
- [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md) - Example backend integration
