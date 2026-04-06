# 38. Authentication & Authorization — OAuth2, JWT, RBAC, ABAC

## What's the Difference?

- **Authentication (AuthN):** "Who are you?" — verifying identity (login)
- **Authorization (AuthZ):** "What can you do?" — verifying permissions (access control)

They are always separate concerns. A valid authenticated user may be unauthorized to access a resource.

---

## Q1: What are the common authentication mechanisms?

### 1. Session-Based Authentication (Stateful)

```
Client → POST /login {credentials}
Server → validates → creates session in DB/Redis
Server → sets cookie: session_id=abc123
Client → sends cookie on every request
Server → looks up session in DB → identifies user
```

**Pros:** Easy to invalidate (delete session from DB), server controls session state
**Cons:** Server must store sessions (memory/Redis), doesn't scale horizontally without shared session store, CSRF vulnerability with cookies

---

### 2. Token-Based Authentication — JWT (Stateless)

```
Client → POST /login {credentials}
Server → validates → signs JWT → returns token
Client → stores token (localStorage/memory)
Client → Authorization: Bearer <token> on every request
Server → verifies JWT signature → no DB lookup needed
```

JWT is self-contained — the server doesn't store any session state.

---

### 3. API Keys

Long-lived opaque strings issued to developers/services.

```
GET /api/data
Authorization: ApiKey sk-prod-abc123def456
```

Server looks up key in DB, checks permissions. Simple but no expiry by default. Rotation must be manual.

**Used by:** Stripe, OpenAI, AWS (for programmatic access)

---

### 4. mTLS (Mutual TLS)

Both client and server present certificates. No passwords or tokens — identity is the certificate.

Used for: service-to-service auth in microservices, zero-trust architectures.

---

## Q2: How does JWT work in depth?

### Structure

JWT = Base64(Header) + "." + Base64(Payload) + "." + Signature

```json
// Header
{ "alg": "RS256", "typ": "JWT" }

// Payload (claims)
{
  "sub": "user_12345",       // subject (user ID)
  "iss": "auth.myapp.com",   // issuer
  "aud": "api.myapp.com",    // audience
  "iat": 1704067200,         // issued at (Unix timestamp)
  "exp": 1704070800,         // expiry (1 hour later)
  "roles": ["user", "admin"],
  "email": "user@example.com"
}

// Signature (RS256)
RSA_SHA256(base64(header) + "." + base64(payload), private_key)
```

**Verification:** Any service with the public key can verify the JWT without a DB call — this is the stateless advantage.

### JWT Signing Algorithms

| Algorithm | Type | Use Case |
|-----------|------|----------|
| **HS256** | Symmetric (shared secret) | Simple, single-service; secret must be shared |
| **RS256** | Asymmetric (RSA) | Auth server signs with private key, services verify with public key |
| **ES256** | Asymmetric (ECDSA) | Smaller keys than RSA, same security |

**Production recommendation:** RS256 or ES256. Never share the private key — only the public key goes to resource servers.

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| Lifetime | Short (5–60 min) | Long (7–90 days) |
| Stored | Memory / JS variable | HttpOnly cookie or secure storage |
| Used for | API calls | Getting new access tokens |
| If stolen | Attacker has 5–60 min | Long-lived risk — must be revoked |

**Flow:**
```
Login → [access_token (15min), refresh_token (7d)]
Access token expires → POST /auth/refresh {refresh_token}
Server validates refresh token (checks DB) → issues new access_token
Logout → invalidate refresh token in DB
```

### JWT Revocation Problem

JWTs are stateless — you can't "un-sign" one. If a user logs out or is compromised:
- **Short expiry** (5–15 min) is the best mitigation
- **Blocklist in Redis:** store revoked JWI (JWT ID) until expiry, check on every request — reintroduces statefulness
- **Refresh token rotation:** Issue new refresh token on each use; old one becomes invalid

---

## Q3: How does OAuth 2.0 work?

OAuth 2.0 is an **authorization framework** — it lets users grant third-party apps limited access to their data without sharing passwords.

### Key Roles
- **Resource Owner:** The user
- **Client:** The third-party application
- **Authorization Server:** Issues tokens (e.g., Google's OAuth server)
- **Resource Server:** The API being accessed (e.g., Google Drive API)

### Authorization Code Flow (most common, most secure)

```
1. User clicks "Login with Google" on MyApp
2. MyApp redirects → Google's Auth Server
   GET /authorize?client_id=xxx&redirect_uri=myapp.com/callback
              &response_type=code&scope=email profile&state=random_csrf_token

3. User logs in to Google, grants permissions
4. Google redirects → myapp.com/callback?code=AUTH_CODE&state=random_csrf_token

5. MyApp server exchanges code for tokens (server-to-server, never in browser):
   POST /token {code, client_id, client_secret, redirect_uri}
   ← {access_token, refresh_token, expires_in}

6. MyApp calls Google API with access_token
   GET https://www.googleapis.com/oauth2/v3/userinfo
   Authorization: Bearer ACCESS_TOKEN
```

**Why the code exchange?** The authorization code is short-lived and single-use. The actual access token is obtained server-side — never exposed in the browser URL where it could be logged or leaked.

### PKCE (Proof Key for Code Exchange)

For mobile apps and SPAs that can't securely store a `client_secret`:

1. App generates a random `code_verifier`
2. App sends `code_challenge = SHA256(code_verifier)` in the initial request
3. When exchanging the code, app sends `code_verifier` — server verifies the hash
4. Even if the auth code is intercepted, it's useless without the verifier

### OAuth Flows Summary

| Flow | Use Case | Security |
|------|----------|---------|
| Authorization Code + PKCE | Web apps, mobile apps | ✅ Most secure |
| Client Credentials | Service-to-service, no user | ✅ Good for M2M |
| Device Code | Smart TVs, CLI tools | ✅ Good |
| Implicit | ❌ Deprecated | ❌ Token in URL |
| Resource Owner Password | ❌ Legacy only | ❌ Client sees password |

---

## Q4: What is RBAC (Role-Based Access Control)?

Users are assigned **roles**. Roles have **permissions**. Access is granted based on role membership.

```
User → assigned → Role → has → Permissions

user_12345 → [Role: admin] → [Permissions: read, write, delete]
user_67890 → [Role: viewer] → [Permissions: read]
```

**DB Schema:**
```sql
users (id, email, ...)
roles (id, name)
permissions (id, resource, action)
user_roles (user_id, role_id)
role_permissions (role_id, permission_id)
```

**Check:**
```python
def can_access(user_id, resource, action):
    return db.query("""
        SELECT 1 FROM user_roles ur
        JOIN role_permissions rp ON ur.role_id = rp.role_id
        JOIN permissions p ON rp.permission_id = p.id
        WHERE ur.user_id = %s AND p.resource = %s AND p.action = %s
    """, user_id, resource, action)
```

**Pros:** Simple to understand and manage, easy to audit ("who has admin?"), works well for hierarchical organizations

**Cons:** Role explosion — large enterprises end up with hundreds of roles. Doesn't handle fine-grained rules like "can only edit their own documents."

---

## Q5: What is ABAC (Attribute-Based Access Control)?

Access decisions are based on **attributes** of the user, resource, and environment — evaluated against **policies**.

```
Policy: "Allow if user.department == resource.department 
         AND user.clearance >= resource.classification
         AND env.time BETWEEN 09:00 AND 18:00"

User attributes: {department: "engineering", clearance: 3, role: "engineer"}
Resource attributes: {department: "engineering", classification: 2, owner: "user_123"}
Environment: {time: "14:30", ip: "10.0.0.1"}

Decision: PERMIT (all conditions met)
```

**Pros:**
- Fine-grained control ("can only edit documents in their department")
- Handles complex, context-aware rules
- No role explosion — policies handle combinations

**Cons:**
- Complex to implement and debug
- Policy evaluation can be slow (must evaluate every attribute)
- Hard to audit ("why was this denied?")

**RBAC vs ABAC:**

| | RBAC | ABAC |
|---|---|---|
| Granularity | Coarse (role-level) | Fine (attribute-level) |
| Complexity | Low | High |
| Flexibility | Low | High |
| Performance | Fast (role lookup) | Slower (policy eval) |
| Use case | Most enterprise apps | Healthcare, gov, multi-tenant SaaS |

**In practice:** Most systems use RBAC for coarse control + ABAC-style checks for fine-grained rules (e.g., "user can only edit their own resources").

---

## Q6: What is OpenID Connect (OIDC)?

OAuth 2.0 handles *authorization* (access to resources). **OIDC** adds *authentication* (identity) on top of OAuth 2.0.

OIDC adds:
- **ID Token** (JWT containing user identity claims: name, email, sub)
- **UserInfo endpoint** (fetch additional user data with access token)
- Standard claims (`sub`, `email`, `name`, `picture`)

**"Login with Google" uses OIDC**, not just OAuth 2.0. Your app receives an ID token to identify the user, and an access token to call Google APIs on their behalf.

---

## Q7: Where should JWTs be stored?

| Location | XSS Risk | CSRF Risk | Notes |
|----------|----------|-----------|-------|
| `localStorage` | ❌ High | ✅ Safe | JS can read → XSS steals token |
| `sessionStorage` | ❌ High | ✅ Safe | Same as localStorage |
| **HttpOnly cookie** | ✅ Safe | ❌ Risk | JS can't read → use CSRF tokens |
| **Memory (JS var)** | ✅ Safe | ✅ Safe | Lost on refresh — use refresh token in HttpOnly cookie |

**Best practice:** Store access token in memory. Store refresh token in HttpOnly, SameSite=Strict cookie. On page load, silently refresh to get a new access token.

---

## Numbers to Remember

| Metric | Value |
|--------|-------|
| Access token lifetime (typical) | 5–60 minutes |
| Refresh token lifetime (typical) | 7–90 days |
| JWT size (typical) | 500 bytes – 2 KB |
| bcrypt rounds for password hashing | 10–12 (production) |
| PKCE code verifier length | 43–128 chars |
| OAuth authorization code lifetime | 10 minutes (spec max) |

---

## Interview Q&A

**Q: How do you handle JWT revocation without sacrificing statelessness?**
A: The cleanest approach is short-lived access tokens (5–15 min) with refresh token rotation. For immediate revocation (logout, account compromise), keep a Redis blocklist of revoked JWT IDs (jti claim) with TTL = token expiry. On each request, check the blocklist — O(1) Redis lookup. This adds minimal overhead while maintaining near-statelessness. For high-security cases (banking), accept full statefulness and validate every token against a session store.

**Q: When would you choose ABAC over RBAC?**
A: RBAC works for most applications with clear user types (admin, editor, viewer). Switch to ABAC when you need resource-level control ("can only edit documents they created"), time/location-based policies ("access only from office IP during business hours"), or multi-tenant SaaS where tenant A's admin shouldn't touch tenant B's data. ABAC adds complexity — only pay that cost when RBAC's granularity is genuinely insufficient.

**Q: What's the difference between OAuth 2.0 and SAML?**
A: OAuth 2.0/OIDC is modern, JSON/JWT-based, designed for APIs and mobile apps. SAML is older, XML-based, designed for enterprise SSO in browser environments. SAML is still dominant in enterprise B2B (Okta, Salesforce, enterprise identity providers). For consumer apps and APIs, use OAuth 2.0/OIDC. If a customer says "we need SSO with our corporate identity provider," that usually means SAML or OIDC depending on their IdP.