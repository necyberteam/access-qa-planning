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

The identity cookie (`SESSaccess_auth`) contains a signed JWT with the user's ACCESS ID:

```
SESSaccess_auth=<signed-jwt>
```

See [drupal-jwt-cookie-spec.md](./drupal-jwt-cookie-spec.md) for complete implementation details.

### JWT Header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "a1b2c3d4e5f67890"
}
```

| Field | Description |
|-------|-------------|
| `alg` | ES256 (ECDSA P-256 + SHA-256) |
| `kid` | Key ID — used by agent to select the correct public key from the issuer's JWKS endpoint |

### JWT Payload

```json
{
  "iss": "https://support.access-ci.org",
  "sub": "jsmith@access-ci.org",
  "iat": 1707000000,
  "exp": 1707064800
}
```

| Claim | Description |
|-------|-------------|
| `iss` | Issuer URL — identifies which site signed this JWT |
| `sub` | ACCESS ID (from CILogon eppn claim, e.g. `jsmith@access-ci.org`) |
| `iat` | Issued at timestamp (Unix epoch) |
| `exp` | Expiration timestamp (18h after `iat`) |

### JWT Signature

- **Algorithm:** ES256 (ECDSA using P-256 curve and SHA-256)
- **Key:** Site-specific EC P-256 private key (each issuer has its own)
- **Verification:** Agent fetches public key from issuer's `/.well-known/jwks.json` endpoint
- **Validation:** Backend MUST verify signature before trusting claims

### Cookie Attributes

```
Set-Cookie: SESSaccess_auth=<jwt>;
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

### Expiration Behavior

The `SESSaccess_auth` cookie uses **rolling expiration**: it is re-issued on every authenticated response, so the 18-hour window resets with each page load. The cookie only expires after 18 hours of *inactivity*.

This differs from `SESSaccesscisso` (fixed expiration set once at login) — see [drupal-jwt-cookie-spec.md § Rolling vs. Fixed Expiration](./drupal-jwt-cookie-spec.md#rolling-vs-fixed-expiration) for the rationale.

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

1. **Generate an EC P-256 key pair** for the site:

```bash
openssl ecparam -name prime256v1 -genkey -noout -out private.pem
openssl ec -in private.pem -pubout -out public.pem
```

2. **After CILogon authentication**, create a signed JWT:

```python
import jwt
import time

# Load once at startup
with open("private.pem", "rb") as f:
    PRIVATE_KEY = f.read()

ISSUER = "https://mysite.access-ci.org"  # unique per site

def create_session_cookie(access_id: str) -> str:
    payload = {
        "iss": ISSUER,
        "sub": access_id,  # e.g., "jsmith@access-ci.org"
        "iat": int(time.time()),
        "exp": int(time.time()) + 64800  # 18 hours
    }
    return jwt.encode(payload, PRIVATE_KEY, algorithm="ES256",
                      headers={"kid": KID})
```

3. **Set the cookie** with proper attributes:

```python
response.set_cookie(
    "SESSaccess_auth",
    value=create_session_cookie(user.access_id),
    domain=".access-ci.org",
    path="/",
    httponly=True,
    secure=True,
    samesite="None"
)
```

> **Note:** For Drupal sites on Pantheon, this is handled by the `AccessAuthCookieSubscriber` event subscriber — see [drupal-jwt-cookie-spec.md](./drupal-jwt-cookie-spec.md).

4. **Publish the public key** at `/.well-known/jwks.json` so the agent can verify tokens (see [JWKS Discovery](#jwks-discovery))

5. **Store the private key** securely — as a platform environment variable (`ACCESS_JWT_PRIVATE_KEY`) or a file on the server. The private key never leaves the issuing site.

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
from jwt import PyJWKClient

# Configure trusted JWKS endpoints — one per issuing site.
# The agent fetches public keys from these endpoints to verify tokens.
TRUSTED_ISSUERS = {
    "https://support.access-ci.org": PyJWKClient(
        "https://support.access-ci.org/.well-known/jwks.json",
        cache_keys=True,
    ),
    "https://allocations.access-ci.org": PyJWKClient(
        "https://allocations.access-ci.org/.well-known/jwks.json",
        cache_keys=True,
    ),
}

def get_acting_user(request) -> str | None:
    """Extract and validate ActingUser from session cookie.

    Returns None if cookie is missing, invalid, or expired.
    This is NOT an error condition - it means anonymous mode.
    """
    cookie = request.cookies.get("SESSaccess_auth")
    if not cookie:
        return None

    # Handle legacy cookie value (just "1" for logged-in flag)
    if cookie == "1":
        return None

    try:
        # Decode header to get kid and iss without verifying signature.
        unverified = jwt.decode(cookie, options={"verify_signature": False})
        issuer = unverified.get("iss")

        # Look up the JWKS client for this issuer.
        jwks_client = TRUSTED_ISSUERS.get(issuer)
        if not jwks_client:
            return None  # Unknown issuer — treat as anonymous

        # Fetch the public key matching the kid in the JWT header.
        signing_key = jwks_client.get_signing_key_from_jwt(cookie)

        # Verify signature, expiration, and issuer.
        payload = jwt.decode(
            cookie,
            signing_key.key,
            algorithms=["ES256"],
            issuer=list(TRUSTED_ISSUERS.keys()),
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

### Asymmetric Key Management (ES256 + JWKS)

JWTs are signed with **ES256** (ECDSA P-256) using asymmetric key pairs. Each issuing site holds its own private key. The agent validates tokens using public keys fetched from each site's JWKS endpoint. **No shared secret exists.**

This approach is recommended by NIST SP 800-63C, RFC 8725, and OWASP for multi-issuer JWT architectures. See [drupal-jwt-cookie-spec.md § Why Asymmetric](./drupal-jwt-cookie-spec.md#why-asymmetric-es256-instead-of-shared-secret-hs256) for the full comparison.

#### Who Needs What

| Service | Role | Has Private Key | Has Public Key |
|---------|------|----------------|----------------|
| ACCESS Drupal sites | Issue cookies | Yes (their own) | Published at JWKS endpoint |
| Other authenticated ACCESS sites (Django, etc.) | Issue cookies | Yes (their own) | Published at JWKS endpoint |
| QA Bot Agent (`qa.access-ci.org`) | Validate cookies | No | Fetches from JWKS endpoints |
| MCP servers | N/A — trusts agent via service auth | No | No |
| JSM proxy | N/A — trusts agent via API key | No | No |
| Dumb hosts (WordPress, static) | N/A — just embeds chatbot widget | No | No |

#### JWKS Discovery

Each issuing site publishes its public key at `/.well-known/jwks.json`. The agent is configured with a list of trusted issuer URLs and fetches their JWKS endpoints to build a key set for verification.

```python
# Agent configuration — trusted issuers
TRUSTED_ISSUERS = {
    "https://support.access-ci.org": "https://support.access-ci.org/.well-known/jwks.json",
    "https://allocations.access-ci.org": "https://allocations.access-ci.org/.well-known/jwks.json",
}
```

Adding a new issuing site requires only adding its URL to the agent's trusted issuers list and redeploying the agent.

#### Key Rotation

Rotation is per-site, requires no cross-team coordination, and has zero downtime:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KEY ROTATION TIMELINE (per site)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  T+0:    Generate new EC key pair                                   │
│          Add new public key to /.well-known/jwks.json               │
│          (keep old public key in JWKS — both are now listed)        │
│                                                                     │
│  T+0:    Update ACCESS_JWT_PRIVATE_KEY to new private key           │
│          New cookies signed with new key                            │
│          Old cookies (signed with old key) still valid              │
│          Agent fetches JWKS, sees both keys, can verify either      │
│                                                                     │
│  T+18h:  All old cookies have expired                               │
│          Remove old public key from /.well-known/jwks.json          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key advantages over shared-secret rotation:**
- No coordination between teams — each site rotates on its own schedule
- No dual-secret validation window logic in the agent — JWKS handles it naturally via `kid` matching
- Compromise of one site's private key does not affect any other site

#### JWKS Endpoint Availability

If a site's JWKS endpoint is temporarily unreachable, the agent uses cached public keys:

| Scenario | Behavior |
|----------|----------|
| JWKS reachable | Fetch and cache public keys (PyJWKClient handles caching automatically) |
| JWKS unreachable, cache valid | Use cached keys — continue normally |
| JWKS unreachable, no cache | Cannot verify tokens from that issuer — treat as anonymous |

**Key principle:** Fail closed for authentication (don't guess), fail open for anonymous Q&A (don't break the chatbot for unauthenticated users just because a JWKS endpoint is down).

### Token Expiration

- **Lifetime:** 18 hours (from `drupal_seamless_cilogon.seamless_cookie_expiration` Drupal state)
- **Expiration type:** Rolling — cookie is re-issued on every authenticated response, so it only expires after 18 hours of inactivity
- **Behavior on expiration:** Agent treats user as anonymous (graceful degradation, not an error)

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

2. **Generate key pairs for each issuing site**
   - Generate EC P-256 key pair per site (`openssl ecparam -name prime256v1 -genkey -noout`)
   - Store private key as platform environment variable
   - Publish public key at `/.well-known/jwks.json`

3. **Update ACCESS sites to issue JWT cookies**
   - Modify authentication handlers
   - Change cookie value from `1` to ES256-signed JWT
   - Update `SameSite` from `Lax` to `None`
   - Add `iss` claim and `kid` header
   - See [drupal-jwt-cookie-spec.md](./drupal-jwt-cookie-spec.md) for implementation details

4. **Update QA Bot frontend** --- DONE
   - [x] Add `credentials: 'include'` to fetch calls
   - [x] Remove client-side `X-Acting-User` header and `acting_user` body field
   - [x] Deprecate `actingUser` prop (retained for backward compat)

5. **Update QA Bot Agent** --- DONE
   - [x] Add JWT validation logic (`src/auth.py`)
   - [x] Extract ActingUser from validated cookie with body fallback
   - [x] Tighten CORS defaults for production
   - [x] Add `PyJWT[crypto]` dependency (replaces `hvac`)
   - [x] Unit tests for auth module (13 tests including algorithm confusion attack)
   - [x] E2E tests through FastAPI (9 tests covering cookie/body/fallback flows)
   - [x] Switch from HS256 to ES256 validation with JWKS discovery (`PyJWKClient`)
   - [x] Configure trusted issuer JWKS URLs (`TRUSTED_JWKS_URLS` env var)
   - [x] Remove Vault dependency (deleted `src/vault.py`, `hvac`, `scripts/vault-init.sh`)
   - [x] Verify `issuer=` parameter passed to `jwt.decode()` so `iss` is validated (not only used for key lookup)

### Rollback Plan

If issues arise during the ES256 migration:

1. **Agent side**: Re-enable `ALLOW_BODY_ACTING_USER=true` (already the default) so sites can fall back to sending identity in the request body while cookie issues are resolved
2. **Site side**: Each site can independently revert its event subscriber to stop issuing `SESSaccess_auth` cookies without affecting other sites — the agent treats missing cookies as anonymous
3. **Full rollback**: If ES256 must be completely reverted, restore `src/vault.py` and HS256 validation from git history, and re-add `JWT_SECRET` / `VAULT_*` env vars. The body fallback path (`acting_user` in request body) continues to work throughout
4. **Gradual rollout**: Changes are independent per-site. Roll out to one site first (e.g. support.access-ci.org), verify cookies and JWKS endpoint work, then enable on additional sites

## Appendix: Example JWT

### Decoded Header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "a1b2c3d4e5f67890"
}
```

### Decoded Payload

```json
{
  "iss": "https://support.access-ci.org",
  "sub": "jsmith@access-ci.org",
  "iat": 1707000000,
  "exp": 1707064800
}
```

## Adapting for Other Programs

This authentication pattern is designed to be transferable to other programs beyond ACCESS-CI. The core requirements are:

1. **Shared parent domain** - All sites must be under a common domain (e.g., `*.myprogram.org`)
2. **Common identity provider** - A way to authenticate users and obtain a canonical user ID
3. **Per-site signing key** - Each site generates its own EC key pair and publishes the public key via JWKS

### Configuration Points

To adapt for another program, modify these values:

| Parameter | ACCESS-CI Value | Your Program |
|-----------|-----------------|--------------|
| Cookie name | `SESSaccess_auth` | `SESS{program}_auth` |
| Cookie domain | `.access-ci.org` | `.{your-domain}` |
| User ID claim | `sub` (ACCESS ID) | `sub` (your canonical ID) |
| Identity provider | CILogon | Your IdP (Okta, Auth0, SAML, etc.) |
| Backend domain | `qa.access-ci.org` | `qa.{your-domain}` |
| Signing algorithm | ES256 | ES256 (recommended) |

### Minimal Implementation Checklist

For a new program to adopt this pattern:

- [ ] **Choose cookie name** - e.g., `SESSmyprogram_auth`
- [ ] **Generate EC key pair per site** - `openssl ecparam -name prime256v1 -genkey -noout`
- [ ] **Deploy QA Bot Agent** - On a subdomain of your shared domain
- [ ] **Configure agent** - Add each site's JWKS URL to trusted issuers
- [ ] **Update sites to issue JWT cookies** - ES256-signed with `iss` and `kid`
- [ ] **Publish JWKS** - Each site serves `/.well-known/jwks.json` with its public key
- [ ] **Set cookie attributes** - `HttpOnly`, `Secure`, `SameSite=None`, `Domain=.{your-domain}`

### Example: Adapting for "MyResearch" Program

```python
# Configuration for MyResearch program
COOKIE_NAME = "SESSmyresearch_auth"
COOKIE_DOMAIN = ".myresearch.org"
ISSUER = "https://portal.myresearch.org"

# Load site's private key (each site has its own)
with open("private.pem", "rb") as f:
    PRIVATE_KEY = f.read()

# JWT payload uses same structure
payload = {
    "iss": ISSUER,
    "sub": "researcher123@myresearch.org",  # Canonical user ID
    "iat": int(time.time()),
    "exp": int(time.time()) + 64800  # 18 hours
}

# Cookie settings
response.set_cookie(
    COOKIE_NAME,
    value=jwt.encode(payload, PRIVATE_KEY, algorithm="ES256",
                     headers={"kid": KID}),
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
