# Security and Authentication

> **Goal**: Understand authentication, authorization, and security patterns for protecting APIs and distributed systems.

---

## 1. Authentication vs Authorization

```
┌─────────────────────────────────────────────────────────────────────┐
│  AUTHENTICATION (AuthN)        │  AUTHORIZATION (AuthZ)             │
│                                │                                    │
│  "Who are you?"                │  "What can you do?"                │
│                                │                                    │
│  • Verify identity             │  • Check permissions               │
│  • Username/password           │  • Roles and policies              │
│  • Tokens, certificates        │  • Access control                  │
│                                │                                    │
│  Happens first                 │  Happens after authentication      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Authentication Methods

### API Keys

```
┌──────────────────────────────────────────────────────────────────────┐
│  Simple, static credential                                           │
│                                                                      │
│  Request:                                                            │
│  GET /api/data                                                       │
│  X-API-Key: sk_live_abc123xyz                                        │
│                                                                      │
│  ✅ Simple to implement ✅                                          │
│  ✅ Good for server-to-server ✅                                    │
│  ❌ No expiration (unless rotated) ❌                               │
│  ❌ Hard to revoke ❌                                               │
│  ❌ Not user-specific ❌                                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Basic Authentication

```
┌─────────────────────────────────────────────────────────────────────┐
│  Username:password encoded in Base64                                │
│                                                                     │
│  Request:                                                           │
│  GET /api/data                                                      │
│  Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=                      │
│                                                                     │
│  ⚠️ Only use over HTTPS! ⚠️                                        │
│  ❌ Credentials sent every request ❌                              │
│  ❌ No token expiration ❌                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Session-Based Authentication

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Login                                                           │
│     POST /login {username, password}                                │
│     Server creates session, returns cookie                          │
│                                                                     │
│  2. Subsequent Requests                                             │
│     Cookie: session_id=abc123                                       │
│     Server looks up session in store (Redis, DB)                    │
│                                                                     │
│  ✅ Can revoke instantly ✅                                        │
│  ✅ Server controls session ✅                                     │
│  ❌ Requires session storage ❌                                    │
│  ❌ Harder to scale (sticky sessions or shared store) ❌           │
│  ❌ Vulnerable to CSRF ❌                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Token-Based Authentication (JWT)

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Login                                                           │
│     POST /login {username, password}                                │
│     Server returns signed JWT                                       │
│                                                                     │
│  2. Subsequent Requests                                             │
│     Authorization: Bearer eyJhbGciOiJIUzI1NiIs...                   │
│     Server validates signature (no lookup needed)                   │
│                                                                     │
│  ✅ Stateless — easy to scale ✅                                   │
│  ✅ Self-contained (includes user info) ✅                         │
│  ❌ Can't revoke until expiration ❌                               │
│  ❌ Token size larger than session ID ❌                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. JWT (JSON Web Tokens)

### Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│  header.payload.signature                                           │
│                                                                     │
│  eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0In0.signature                 │
│  └───────┬────────┘ └────────┬────────┘ └────┬────┘                 │
│       Header             Payload          Signature                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Header

```json
{
  "alg": "HS256",    // Algorithm (HS256, RS256, ES256)
  "typ": "JWT"
}
```

### Payload (Claims)

```json
{
  // Registered claims
  "sub": "user123",           // Subject (user ID)
  "iss": "auth.example.com",  // Issuer
  "aud": "api.example.com",   // Audience
  "exp": 1609459200,          // Expiration
  "iat": 1609455600,          // Issued at
  "nbf": 1609455600,          // Not before
  
  // Custom claims
  "roles": ["user", "admin"],
  "email": "user@example.com"
}
```

### Signing Algorithms

| Algorithm | Type | Use Case |
|-----------|------|----------|
| **HS256** | Symmetric | Single service, simple |
| **RS256** | Asymmetric | Multiple services, key rotation |
| **ES256** | Asymmetric | Smaller tokens, modern choice |

### JWT Validation

```python
def validate_jwt(token, secret):
    try:
        payload = jwt.decode(
            token,
            secret,
            algorithms=["HS256"],
            audience="api.example.com",
            issuer="auth.example.com"
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise AuthError("Token expired")
    except jwt.InvalidTokenError:
        raise AuthError("Invalid token")
```

### JWT Best Practices

```
✅ Use short expiration (15 min - 1 hour)
✅ Use refresh tokens for long sessions
✅ Validate all claims (exp, iss, aud)
✅ Use asymmetric keys for distributed systems
✅ Store tokens securely (HttpOnly cookies or secure storage)

❌ Don't store sensitive data in payload (it's readable)
❌ Don't use "none" algorithm
❌ Don't accept tokens from untrusted issuers
```

---

## 4. OAuth 2.0

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│  OAuth 2.0 = Authorization framework for third-party access         │
│                                                                     │
│  "Allow App X to access your data on Service Y"                     │
│  Without sharing your password                                      │
│                                                                     │
│  Example: "Sign in with Google"                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### OAuth Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | User who owns the data | You |
| **Client** | App requesting access | Third-party app |
| **Authorization Server** | Issues tokens | Google Auth |
| **Resource Server** | Holds protected data | Google APIs |

### Authorization Code Flow (Most Secure)

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. User clicks "Login with Google"                                 │
│                                                                     │
│  2. Redirect to Google:                                             │
│     /authorize?client_id=xxx&redirect_uri=xxx&scope=email           │
│                                                                     │
│  3. User authenticates with Google                                  │
│                                                                     │
│  4. Google redirects back with code:                                │
│     /callback?code=abc123                                           │
│                                                                     │
│  5. Server exchanges code for tokens:                               │
│     POST /token {code, client_secret}                               │
│     → {access_token, refresh_token}                                 │
│                                                                     │
│  6. Use access_token to call APIs                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### OAuth Grant Types

| Grant | Use Case | Security |
|-------|----------|----------|
| **Authorization Code** | Server-side apps | Most secure |
| **Authorization Code + PKCE** | Mobile/SPA apps | Secure |
| **Client Credentials** | Machine-to-machine | No user |
| **Refresh Token** | Get new access token | Extends session |

> **Note**: Implicit and Password grants are deprecated. Use Authorization Code + PKCE for all client apps.

---

## 5. OpenID Connect (OIDC)

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│  OIDC = OAuth 2.0 + Identity Layer                                  │
│                                                                     │
│  OAuth: "App can access your photos"                                │
│  OIDC:  "App knows who you are"                                     │
│                                                                     │
│  Adds:                                                              │
│  • ID Token (JWT with user info)                                    │
│  • UserInfo endpoint                                                │
│  • Standard claims (name, email, picture)                           │
└─────────────────────────────────────────────────────────────────────┘
```

### ID Token vs Access Token

| Token | Purpose | Format |
|-------|---------|--------|
| **ID Token** | User identity | JWT (always) |
| **Access Token** | API authorization | JWT or opaque |

---

## 6. Authorization Patterns

### Role-Based Access Control (RBAC)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Users have ROLES                                                   │
│  Roles have PERMISSIONS                                             │
│                                                                     │
│  User: alice@example.com                                            │
│  Roles: [editor, viewer]                                            │
│                                                                     │
│  editor role: [create_post, edit_post, delete_post]                 │
│  viewer role: [view_post]                                           │
│                                                                     │
│  Check:                                                             │
│  if "editor" in user.roles:                                         │
│      allow_edit()                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Attribute-Based Access Control (ABAC)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Policies based on attributes                                       │
│                                                                     │
│  Policy: "Users can edit documents in their department"             │
│                                                                     │
│  Check:                                                             │
│  if (user.department == document.department                         │
│      and user.level >= 3                                            │
│      and current_time.is_business_hours()):                         │
│      allow_edit()                                                   │
│                                                                      │
│  More flexible than RBAC, more complex                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Permission-Based

```python
# Direct permission checks
@require_permission("posts:create")
def create_post(data):
    ...

@require_permission("posts:delete")
def delete_post(post_id):
    ...
```

---

## 7. Token Refresh Strategy

### Access + Refresh Tokens

```
┌─────────────────────────────────────────────────────────────────────┐
│  Access Token                    Refresh Token                      │
│  • Short-lived (15 min)          • Long-lived (7 days)              │
│  • Sent with every request       • Used to get new access token     │
│  • Can't revoke easily           • Can be revoked (stored in DB)    │
│                                                                     │
│  Flow:                                                              │
│  1. Login → access_token + refresh_token                            │
│  2. Use access_token for API calls                                  │
│  3. Access token expires                                            │
│  4. Use refresh_token to get new access_token                       │
│  5. If refresh_token invalid → re-login                             │
└─────────────────────────────────────────────────────────────────────┘
```

### Sliding Session

```
┌─────────────────────────────────────────────────────────────────────┐
│  Every successful request extends session                           │
│                                                                     │
│  Access: 15 min, Refresh: 7 days                                    │
│                                                                     │
│  User active → keeps getting new access tokens                      │
│  User inactive for 15 min → needs refresh                           │
│  User inactive for 7 days → must re-login                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Token Rotation

```python
# When refresh token is used, issue new refresh token
def refresh_tokens(refresh_token):
    if not is_valid(refresh_token):
        raise InvalidToken()
    
    # Invalidate old refresh token
    revoke(refresh_token)
    
    # Issue new tokens
    new_access = create_access_token(user)
    new_refresh = create_refresh_token(user)
    
    return new_access, new_refresh
```

---

## 8. API Security

### HTTPS Everywhere

```
✅ All traffic over TLS
✅ HSTS header (force HTTPS)
✅ Certificate validation
```

### Rate Limiting

```python
# Protect against brute force and DoS
@rate_limit("100/minute")
def login(credentials):
    ...

@rate_limit("1000/minute")
def api_call():
    ...

# Response headers
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1609459200
```

### Input Validation

```python
# Never trust user input
def create_user(data):
    # Validate and sanitize
    email = validate_email(data.get("email"))
    name = sanitize_string(data.get("name"), max_length=100)
    
    # Parameterized queries (prevent SQL injection)
    db.execute("INSERT INTO users (email, name) VALUES (?, ?)", email, name)
```

### Security Headers

```http
# Prevent clickjacking
X-Frame-Options: DENY

# Prevent MIME sniffing
X-Content-Type-Options: nosniff

# XSS protection
Content-Security-Policy: default-src 'self'

# Force HTTPS
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### CORS (Cross-Origin Resource Sharing)

```python
# Control which origins can access your API
CORS_ORIGINS = [
    "https://app.example.com",
    "https://admin.example.com"
]

# Response headers
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
```

---

## 9. Password Security

### Hashing (Never Store Plain Passwords)

```python
import bcrypt

# Hash password
def hash_password(password: str) -> str:
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(password.encode(), salt).decode()

# Verify password
def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode(), hashed.encode())
```

### Password Requirements

```
✅ Minimum 8 characters (12+ recommended)
✅ Check against common password lists
✅ Allow spaces and special characters
✅ No maximum length (let bcrypt handle it)

❌ Don't require specific character types (annoying, marginal benefit)
❌ Don't expire passwords (encourages weak passwords)
```

### Multi-Factor Authentication (MFA)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Something you KNOW + Something you HAVE + Something you ARE        │
│                                                                     │
│  Password + TOTP App + Fingerprint                                  │
│                                                                     │
│  Common MFA methods:                                                │
│  • TOTP (Authenticator apps)                                        │
│  • SMS codes (less secure)                                          │
│  • Push notifications                                               │
│  • Hardware keys (FIDO2/WebAuthn)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. Secrets Management

### Never in Code

```python
# ❌ Never do this
API_KEY = "sk_live_abc123"
DATABASE_URL = "postgresql://user:password@host/db"

# ✅ Use environment variables
API_KEY = os.getenv("API_KEY")
DATABASE_URL = os.getenv("DATABASE_URL")
```

### Secrets Management Tools

| Tool | Type | Use Case |
|------|------|----------|
| **HashiCorp Vault** | Self-hosted | Enterprise, complex needs |
| **AWS Secrets Manager** | Cloud | AWS environments |
| **GCP Secret Manager** | Cloud | GCP environments |
| **Azure Key Vault** | Cloud | Azure environments |
| **Doppler** | SaaS | Simple, multi-cloud |

### Rotation

```
✅ Rotate secrets regularly
✅ Support multiple active secrets during rotation
✅ Automate rotation when possible
✅ Audit secret access
```

---

## 11. Common Vulnerabilities

### Broken Authentication

```
❌ Weak passwords allowed
❌ No brute force protection
❌ Session fixation
❌ Credential stuffing

✅ Strong password policies
✅ Rate limiting on login
✅ Regenerate session on login
✅ Detect compromised credentials
```

### Broken Access Control

```
❌ Insecure direct object references
   /api/users/123 → Can access any user?

❌ Missing function-level access control
   Admin endpoints without auth check

✅ Always verify ownership/permissions
✅ Deny by default, allow explicitly
```

### Injection

```python
# ❌ SQL Injection
query = f"SELECT * FROM users WHERE id = {user_input}"

# ✅ Parameterized query
query = "SELECT * FROM users WHERE id = ?"
cursor.execute(query, (user_input,))
```

---

## 12. Security Checklist

```
Authentication:
□ HTTPS only
□ Secure password hashing (bcrypt, argon2)
□ Rate limiting on auth endpoints
□ MFA option
□ Secure session management
□ Token expiration and refresh

Authorization:
□ Check permissions on every request
□ Deny by default
□ Validate object ownership
□ Audit logging

API Security:
□ Input validation
□ Output encoding
□ Security headers
□ CORS configuration
□ Rate limiting

Secrets:
□ No secrets in code
□ Environment variables or secrets manager
□ Regular rotation
□ Audit access
```

---

## TL;DR

| Method | Best For |
|--------|----------|
| **API Keys** | Server-to-server, simple integrations |
| **Sessions** | Traditional web apps, need instant revocation |
| **JWT** | Stateless APIs, microservices |
| **OAuth 2.0** | Third-party access, "Login with X" |
| **OIDC** | User identity + OAuth |

**Key Principles:**
- Authentication = who are you; Authorization = what can you do
- Use JWTs with short expiration + refresh tokens
- OAuth 2.0 for third-party access, OIDC for identity
- Always HTTPS, always hash passwords
- Never store secrets in code

---

## Quick Reference

### JWT Creation

```python
import jwt
from datetime import datetime, timedelta

def create_token(user_id, secret):
    payload = {
        "sub": user_id,
        "exp": datetime.utcnow() + timedelta(hours=1),
        "iat": datetime.utcnow()
    }
    return jwt.encode(payload, secret, algorithm="HS256")
```

### FastAPI Auth Example

```python
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

@app.get("/protected")
async def protected(user: User = Depends(get_current_user)):
    return {"user": user}
```

### Interview Talking Points

1. "JWT is stateless—the server doesn't need to store sessions, but tokens can't be revoked until expiration"
2. "Use short-lived access tokens with refresh tokens for the best balance of security and UX"
3. "OAuth 2.0 is for authorization (access), OIDC adds identity (who you are)"
4. "Always use parameterized queries to prevent SQL injection"
5. "Hash passwords with bcrypt or argon2; never store plain text"
