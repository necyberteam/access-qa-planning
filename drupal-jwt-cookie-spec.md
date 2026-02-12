# Drupal JWT Cookie Implementation Spec

> **Audience:** Developer implementing the OIDC rewrite of `drupal_seamless_cilogon`
>
> **Status:** Implemented — `access/src/EventSubscriber/AccessAuthCookieSubscriber.php`
>
> **Parent doc:** [08-qa-bot-authentication.md](./08-qa-bot-authentication.md)

## Purpose

Set a signed JWT cookie (`SESSaccess_auth`) on `.access-ci.org` whenever a user is authenticated. The QA Bot agent validates this cookie server-side to determine user identity — replacing the current insecure client-side `X-Acting-User` header.

## Relationship to Existing Cookies

The `drupal_seamless_cilogon` module already manages the `SESSaccesscisso` cookie. The new `SESSaccess_auth` cookie is **separate and complementary**:

| Cookie | Purpose | Contains Identity | Expiration | Expiration Behavior | Managed By |
|--------|---------|-------------------|------------|---------------------|------------|
| `SESSaccesscisso` | Seamless SSO redirect signal | No (site name string) | 18 hours (configurable) | **Fixed** — set once at login | `drupal_seamless_cilogon` |
| `SESSaccess_auth` (new) | Signed user identity for agent | Yes (JWT with ACCESS ID) | 18 hours (same setting) | **Rolling** — refreshed every response | `access` module |

Both cookies use the **same** `.access-ci.org` domain and the **same** 18-hour TTL (from `drupal_seamless_cilogon.seamless_cookie_expiration` Drupal state). However, their expiration behavior differs — see [Rolling vs. Fixed Expiration](#rolling-vs-fixed-expiration) below.

**Do not modify or replace `SESSaccesscisso`** — it has its own lifecycle managed by the SSO module.

## Implementation

**File:** `access/src/EventSubscriber/AccessAuthCookieSubscriber.php`

The event subscriber listens on `KernelEvents::RESPONSE` (priority 0). On every main request:

- **Authenticated user:** Sets `SESSaccess_auth` cookie with a signed JWT
- **Anonymous user:** Clears `SESSaccess_auth` cookie (handles logout)

### Key Design Decisions

1. **Expiration reads from `drupal_seamless_cilogon.seamless_cookie_expiration`** (default: `'+18 hours'`) — same Drupal state key used by `SESSaccesscisso`. Both cookies stay in sync.

2. **Domain reads from `drupal_seamless_cilogon.seamless_cookie_domain`** (default: `.access-ci.org`) — same domain for both cookies.

3. **`sub` claim uses the full account name** (e.g., `jsmith@access-ci.org`), not the stripped `accessId` (`jsmith`). This matches the format MCP servers expect in the `X-Acting-User` header.

4. **Placed in `access` module** (not `access_misc` or `drupal_seamless_cilogon`) because the JWT cookie is a core identity concern directly tied to the user data extraction in `access.module:78-104`.

5. **ES256 (asymmetric) instead of HS256 (shared secret).** Each issuing site holds its own EC P-256 private key. The agent validates using public keys fetched from JWKS endpoints. This eliminates shared secret distribution across multiple services and teams (see [Key Management](#key-management)).

### Rolling vs. Fixed Expiration

The two cookies intentionally use different expiration strategies:

- **`SESSaccesscisso` (Fixed):** Set once during CILogon login; the expiration timestamp never changes. After 18 hours the cookie expires regardless of user activity. This is appropriate for SSO — re-authentication is a deliberate security gate.

- **`SESSaccess_auth` (Rolling):** Refreshed on every authenticated response via the `KernelEvents::RESPONSE` subscriber. The 18-hour window resets with each page load, so the cookie only expires after 18 hours of *inactivity*. This is better UX for long working sessions — a researcher actively using the site should not lose their identity cookie (and QA Bot access) mid-session.

**Practical effect:** A user who logs in and works continuously for 20 hours will lose `SESSaccesscisso` at the 18-hour mark (forcing SSO re-auth) but their `SESSaccess_auth` cookie will still be valid. On re-auth, both cookies are reset. This is the desired behavior — re-auth refreshes the SSO signal, while the identity cookie was never stale because the user was active.

### Service Registration

**File:** `access/access.services.yml`

```yaml
services:
  access.auth_cookie_subscriber:
    class: Drupal\access\EventSubscriber\AccessAuthCookieSubscriber
    tags:
      - { name: event_subscriber }
```

### Dependency

**File:** `access/composer.json`

```
"firebase/php-jwt": "^6.0"
```

## Cookie Specification

### Name
`SESSaccess_auth`

### Attributes

| Attribute | Value | Notes |
|-----------|-------|-------|
| `Domain` | `.access-ci.org` | From `drupal_seamless_cilogon.seamless_cookie_domain` state |
| `Path` | `/` | Available to all paths |
| `HttpOnly` | `true` | XSS protection — JS cannot read |
| `Secure` | `true` | HTTPS only |
| `SameSite` | `None` | Required for cross-subdomain AJAX |
| `Expires` | 18 hours from *last response* (rolling) | Refreshed on every authenticated response; from `drupal_seamless_cilogon.seamless_cookie_expiration` state |

### JWT Header

| Field | Value | Description |
|-------|-------|-------------|
| `alg` | `ES256` | ECDSA using P-256 curve and SHA-256 |
| `typ` | `JWT` | Token type |
| `kid` | e.g. `a1b2c3d4e5f67890` | SHA-256 thumbprint of the public key (first 16 hex chars); used by the agent to select the correct key from the JWKS endpoint |

### JWT Claims

| Claim | Type | Description | Example |
|-------|------|-------------|---------|
| `iss` | string | Issuer URL — identifies which site signed this JWT | `"https://support.access-ci.org"` |
| `sub` | string | Full account name (from `getAccountName()`) | `"jsmith@access-ci.org"` |
| `iat` | integer | Issued-at timestamp (Unix epoch) | `1707000000` |
| `exp` | integer | Expiration timestamp (18h after `iat`) | `1707064800` |

### JWT Signing

- **Algorithm:** ES256 (ECDSA P-256 + SHA-256)
- **Key:** Site-specific EC private key (see [Key Management](#key-management))
- **Library:** `firebase/php-jwt` ^6.0

## Key Management

Each issuing site holds its own EC P-256 private key. The agent validates tokens using the corresponding public key published at the site's JWKS endpoint. **No shared secret exists** — compromising one site's key does not affect any other site.

### Why Asymmetric (ES256) Instead of Shared Secret (HS256)

| Concern | HS256 (shared secret) | ES256 (asymmetric) |
|---------|----------------------|-------------------|
| Secret distribution | Same secret on every issuer + validator | Each site has its own private key; agent only needs public keys |
| Blast radius | Compromise of any site exposes all sites | Compromise isolated to one site |
| Rotation coordination | All services must update simultaneously | Each site rotates independently |
| Validator can forge tokens | Yes (holds the same secret) | No (only has public key) |
| Standards alignment | NIST 800-63C requires per-RP keys if HMAC | Explicitly recommended for multi-issuer |

### Generating a Key Pair

```bash
# Generate EC P-256 private key
openssl ecparam -name prime256v1 -genkey -noout -out private.pem

# Extract public key
openssl ec -in private.pem -pubout -out public.pem
```

### Configuring the Private Key

The subscriber loads the private key from environment variables:

| Variable | Format | Example |
|----------|--------|---------|
| `ACCESS_JWT_PRIVATE_KEY` | PEM string (inline) | `-----BEGIN EC PRIVATE KEY-----\nMHQC...` |
| `ACCESS_JWT_PRIVATE_KEY_FILE` | File path to PEM | `/var/private/jwt-signing.pem` |

The subscriber tries `ACCESS_JWT_PRIVATE_KEY` first, then `ACCESS_JWT_PRIVATE_KEY_FILE`. On Pantheon, use the dashboard environment variables to set `ACCESS_JWT_PRIVATE_KEY`. For DDEV, either works.

### Publishing the Public Key (JWKS)

Each issuing site publishes its public key at `/.well-known/jwks.json`. The agent fetches this endpoint to get the public key for signature verification.

Example JWKS response:

```json
{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
      "kid": "a1b2c3d4e5f67890",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

This can be served as a static JSON file or via a Drupal route. The `kid` must match the `kid` in JWT headers signed by this site.

During key rotation, include both old and new public keys in the JWKS response. After the token TTL (18 hours), remove the old key.

### Key Rotation

Rotation is per-site and requires no cross-team coordination:

1. Generate a new key pair
2. Add the new public key to `/.well-known/jwks.json` (keep the old one)
3. Update `ACCESS_JWT_PRIVATE_KEY` to the new private key
4. Wait 18 hours for all old tokens to expire
5. Remove the old public key from `/.well-known/jwks.json`

## Logout Cleanup

Handled automatically by the event subscriber. When a user becomes anonymous (after logout), `clearAuthCookie()` is called on the response, setting the cookie expiration in the past.

## Testing Checklist

### Manual Testing

- [ ] **Login:** After CILogon login, verify `SESSaccess_auth` cookie appears in browser DevTools (Application > Cookies)
- [ ] **Cookie attributes:** Verify Domain=`.access-ci.org`, HttpOnly=true, Secure=true, SameSite=None
- [ ] **JWT contents:** Decode the cookie value at [jwt.io](https://jwt.io) — verify `alg` is `ES256`, `kid` is present in header, `iss` and `sub` are correct in payload
- [ ] **Expiration:** Verify `exp` claim is ~18h after `iat` (matching `SESSaccesscisso` expiration)
- [ ] **Logout:** After logout, verify `SESSaccess_auth` cookie is deleted
- [ ] **Anonymous:** Verify no `SESSaccess_auth` cookie is set for anonymous visitors
- [ ] **Coexistence:** Verify `SESSaccesscisso` cookie is unaffected (still present, correct value)

### Integration Testing

- [ ] **Agent receives cookie:** With QA Bot on an ACCESS site, verify the `SESSaccess_auth` cookie is sent in network requests to the agent (DevTools > Network > request > Cookies)
- [ ] **Agent validates:** Check agent logs for `acting_user=jsmith@access-ci.org` (not `anonymous`)
- [ ] **JWKS endpoint:** Verify `/.well-known/jwks.json` returns the correct public key with matching `kid`
- [ ] **Expired cookie:** Wait for expiration (or set short TTL for testing), verify agent treats user as anonymous

### Edge Cases

- [ ] **User without ACCESS ID:** If account name doesn't end in `@access-ci.org`, no cookie should be set (no error)
- [ ] **Missing private key:** If neither `ACCESS_JWT_PRIVATE_KEY` nor `ACCESS_JWT_PRIVATE_KEY_FILE` is set, no cookie should be set (logs warning once, no error)
- [ ] **Invalid private key:** If the env var contains a non-EC key (e.g., RSA), logs warning, no cookie set
- [ ] **Multiple tabs:** Cookie should be shared across tabs (same browser)

## DNS Prerequisite

The cookie won't flow cross-origin until the agent is deployed on `*.access-ci.org`. The code is backward-compatible — setting the cookie has no side effects before DNS migration.

## Related Files

| File | Purpose |
|------|---------|
| `access/src/EventSubscriber/AccessAuthCookieSubscriber.php` | Sets/clears the JWT cookie |
| `access/access.services.yml` | Registers the event subscriber |
| `access.module:78-104` | Existing user data extraction (same `getAccountName()` source) |
| `drupal_seamless_cilogon/src/EventSubscriber/DrupalSeamlessCilogonEventSubscriber.php` | Reference: `SESSaccesscisso` cookie pattern |
| `access-agent/src/auth.py` | Agent-side JWT validation (consumer of this cookie) |
| `access-qa-planning/08-qa-bot-authentication.md` | Full authentication architecture |
