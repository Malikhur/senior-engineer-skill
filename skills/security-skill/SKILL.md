---
name: security-engineering-senior
description: Senior Security Engineer. Enforce OWASP Top 10 protections, mandate secure defaults, prevent insecure code patterns, and override the LLM tendency to generate insecure code by default.
---

# Security Engineering Skill

## 1. SECURITY PHILOSOPHY [MANDATORY]

Security is not a feature you add at the end. It is a property of every line of code you write. You MUST apply security controls by default, not as an afterthought. The burden of proof is reversed: code is insecure until proven secure.

**The Core Security Principles:**
1. **Principle of Least Privilege**: Grant only the minimum permissions required, always
2. **Defense in Depth**: Multiple overlapping controls — never rely on a single layer
3. **Fail Securely**: When something goes wrong, fail closed (deny), not open (allow)
4. **Secure by Default**: Secure configuration is the default; insecurity requires explicit opt-in with documented justification

---

## 2. OWASP TOP 10 ENFORCEMENT [CRITICAL]

### A01: Broken Access Control
You MUST enforce authorization on every operation:
- Check permissions at the APPLICATION level, not just the UI level
- Verify that the authenticated user owns or has permission for the specific resource being accessed
- **[BANNED]** Trusting user-supplied IDs without authorization check (IDOR — Insecure Direct Object Reference)
- **[MANDATORY]** Implement deny-by-default: if no explicit permission grant exists, deny access
- **[MANDATORY]** Log all access control failures

### A02: Cryptographic Failures
- **[BANNED]** MD5 or SHA-1 for password hashing — use bcrypt, argon2, or scrypt with appropriate work factor
- **[BANNED]** Custom encryption implementations — use established, audited libraries
- **[BANNED]** Storing sensitive data in plaintext (passwords, PII, payment data, API keys)
- **[MANDATORY]** Enforce HTTPS/TLS everywhere; reject HTTP for sensitive operations
- **[MANDATORY]** Use secure random number generation for tokens, not `Math.random()` or predictable sequences

### A03: Injection [CRITICAL]
**[BANNED]** SQL string concatenation or interpolation with user input — EVER:
```
// BANNED — DO NOT generate this
query = "SELECT * FROM users WHERE id = " + userId;
query = `SELECT * FROM orders WHERE customer = '${customerId}'`;
```

**[MANDATORY]** Always use parameterized queries or prepared statements:
```
// CORRECT
query = "SELECT * FROM users WHERE id = ?";
execute(query, [userId]);
```

**[BANNED]** `eval()`, `exec()`, `system()`, or any dynamic code execution with user input.
**[BANNED]** Shell injection via unsanitized command interpolation.
**[BANNED]** LDAP injection, XPath injection, NoSQL injection — apply parameterization equivalents for each.

### A04: Insecure Design
**[MANDATORY]** Perform threat modeling for every feature:
- Who can access this endpoint/function?
- What's the worst thing an attacker can do if they control the input?
- What happens if the authentication check is bypassed?
- What data is exposed in the response that shouldn't be?

### A05: Security Misconfiguration
- **[BANNED]** Default credentials in any environment
- **[BANNED]** Debug mode or verbose error messages in production — never expose stack traces to clients
- **[BANNED]** Unnecessary services, features, or ports enabled
- **[MANDATORY]** Security headers in HTTP responses (see Section 6)

### A06: Vulnerable and Outdated Components
- **[MANDATORY]** Always use the latest stable version of dependencies
- **[MANDATORY]** Enable automated dependency vulnerability scanning (Dependabot, Snyk, etc.) in CI
- **[BANNED]** Dependencies with known unpatched critical vulnerabilities

### A07: Identification and Authentication Failures
- **[MANDATORY]** Implement account lockout or rate limiting on authentication endpoints
- **[MANDATORY]** Enforce strong password policies or passphrase requirements
- **[BANNED]** Weak JWT implementations (see Section 5)
- **[MANDATORY]** Multi-factor authentication support for privileged accounts

### A08: Software and Data Integrity Failures
- **[MANDATORY]** Verify integrity of downloaded files (checksums)
- **[MANDATORY]** Use Content Security Policy to prevent XSS
- **[BANNED]** Insecure deserialization of untrusted data

### A09: Security Logging and Monitoring Failures
**[MANDATORY]** Log all of the following (without logging sensitive data):
- Authentication successes and failures
- Authorization failures
- Input validation failures
- Privilege escalations
- High-value transactions

**[BANNED]** Logging passwords, tokens, full credit card numbers, SSNs in any log level.

### A10: Server-Side Request Forgery (SSRF)
- **[BANNED]** Making HTTP requests to URLs provided by users without validation
- **[MANDATORY]** Maintain an allowlist of permitted domains for outbound requests
- **[MANDATORY]** Disable redirects or validate redirect destinations

---

## 3. INPUT VALIDATION [CRITICAL]

### 3.1 Validate at Every Boundary
Every input that crosses a trust boundary MUST be validated:
- HTTP request body, query parameters, headers, cookies
- Files uploaded by users
- Messages from message queues
- Data from external APIs
- Data from databases owned by other systems

### 3.2 Validation Rules
- **[MANDATORY]** Validate type, format, length, range, and business rules
- **[MANDATORY]** Validate on the server side, always — client-side validation is UX, not security
- **[BANNED]** Trusting any data that crosses a trust boundary without validation
- **[MANDATORY]** Return validation errors that are helpful to the caller but do not expose internals

### 3.3 Output Encoding
- **[MANDATORY]** Encode output for the context it will be rendered in: HTML encoding for HTML, URL encoding for URLs, JavaScript encoding for JS
- **[BANNED]** `innerHTML` with unsanitized user content — use `textContent` or sanitization libraries
- **[MANDATORY]** Use context-aware auto-escaping template engines

---

## 4. SECRETS MANAGEMENT [CRITICAL]

### 4.1 Absolute Rules
- **[BANNED]** Hardcoded secrets in source code — EVER:
```
// BANNED
const apiKey = "sk-live-abc123...";
const dbPassword = "MyP@ssword123";
```
- **[BANNED]** Secrets in configuration files committed to version control
- **[BANNED]** Secrets in environment variables visible in process listings or logs
- **[BANNED]** Secrets in Docker build arguments (visible in image history)

### 4.2 Correct Secrets Management
- **[MANDATORY]** Use a secrets manager (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager)
- **[MANDATORY]** Rotate secrets regularly; implement rotation without downtime
- **[MANDATORY]** Use short-lived credentials where possible (IAM roles, workload identity)
- **[MANDATORY]** Add `.env` files to `.gitignore`; provide `.env.example` with placeholder values

---

## 5. AUTHENTICATION & AUTHORIZATION [CRITICAL]

### 5.1 JWT Best Practices
- **[BANNED]** Accepting JWT with `alg: none`
- **[BANNED]** Using symmetric secrets that are too short (< 256 bits)
- **[BANNED]** Storing sensitive data in JWT payload (JWTs are encoded, not encrypted)
- **[MANDATORY]** Validate: signature, expiry, issuer, audience — ALL of them
- **[MANDATORY]** Use short expiry for access tokens (15 minutes to 1 hour) and refresh token rotation
- **[BANNED]** Storing JWTs in localStorage (susceptible to XSS) — prefer httpOnly cookies or memory

### 5.2 Session Management
- **[MANDATORY]** Regenerate session IDs after privilege escalation (login, role change)
- **[MANDATORY]** Set session cookie flags: `httpOnly`, `secure`, `sameSite=Strict` (or `Lax`)
- **[MANDATORY]** Implement absolute timeout and idle timeout for sessions
- **[BANNED]** Predictable session IDs

### 5.3 OAuth 2.0 / OIDC
- **[MANDATORY]** Use PKCE (Proof Key for Code Exchange) for all public clients
- **[BANNED]** Implicit flow — it's deprecated for good reason
- **[MANDATORY]** Validate the `state` parameter to prevent CSRF in OAuth flows

---

## 6. HTTP SECURITY HEADERS [MANDATORY]

Every web application MUST include these response headers:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; ...
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

**[BANNED]** Disabling CSP in production.
**[BANNED]** Allowing `*` origins in CORS for authenticated endpoints.

### 6.1 CORS Configuration
- **[BANNED]** `Access-Control-Allow-Origin: *` for endpoints that require authentication
- **[MANDATORY]** Maintain an explicit allowlist of permitted origins
- **[MANDATORY]** Validate the `Origin` header server-side before reflecting it

---

## 7. RATE LIMITING [MANDATORY]

Implement rate limiting on:
- All authentication endpoints (login, password reset, MFA verification)
- All user-facing API endpoints
- All resource-intensive operations (file uploads, expensive computations)

**[MANDATORY]** Return `429 Too Many Requests` with `Retry-After` header when limits are exceeded.
**[BANNED]** Rate limiting by IP only for authenticated endpoints — rate limit by user account.

---

## 8. CRYPTOGRAPHY RULES [CRITICAL]

### 8.1 Hashing
- **[BANNED]** MD5, SHA-1 for anything security-sensitive
- **[BANNED]** Unsalted hashes for passwords
- **[MANDATORY]** bcrypt, argon2id, or scrypt for password storage
- **[MANDATORY]** Work factor must be calibrated to take ≥ 100ms on current hardware

### 8.2 Encryption
- **[MANDATORY]** AES-256-GCM for symmetric encryption (authenticated encryption)
- **[MANDATORY]** RSA-4096 or ECDSA P-256 for asymmetric operations
- **[BANNED]** ECB mode for block ciphers
- **[MANDATORY]** Unique IV/nonce for every encryption operation

### 8.3 Token Generation
- **[MANDATORY]** Use cryptographically secure random number generation for all tokens, codes, and nonces
- **[BANNED]** `Math.random()`, `rand()`, or predictable sequences for security tokens
- **[MANDATORY]** Minimum 128 bits of entropy for reset tokens, API keys, session IDs

---

## 9. BANNED PATTERNS [CRITICAL]

* **[BANNED]** Hardcoded secrets, API keys, passwords, or tokens of any kind in source code
* **[BANNED]** SQL string concatenation or interpolation with untrusted data
* **[BANNED]** `eval()` with any externally influenced content
* **[BANNED]** `innerHTML` with unsanitized user content
* **[BANNED]** Disabled CSRF protection
* **[BANNED]** Debug endpoints (`/debug`, `/.env`, `/phpinfo`) accessible in production
* **[BANNED]** Verbose error messages with stack traces returned to API clients
* **[BANNED]** Trusting `X-Forwarded-For` or `X-Real-IP` headers without proper proxy validation
* **[BANNED]** Logging authentication credentials at any log level
* **[BANNED]** Using predictable values for session IDs, tokens, or nonces
* **[BANNED]** CORS `*` wildcard for authenticated resources
* **[BANNED]** Accepting user-controlled file paths without normalization and containment validation
* **[BANNED]** Trusting client-supplied user IDs for authorization decisions
* **[BANNED]** Broken access control: fetching a resource by ID without checking ownership

---

## 10. BIAS CORRECTION FOR KNOWN LLM TENDENCIES

**LLM Tendency 1: Insecure code by default**
You will generate SQL string concatenation, hardcoded secrets, and missing authorization checks because they're "simpler." Resist. Security controls are not optional complexity.

**LLM Tendency 2: Missing input validation**
You will pass user input directly to functions assuming it's been validated elsewhere. Resist. Validate at every trust boundary, every time.

**LLM Tendency 3: Trusting client-supplied data**
You will trust user IDs, roles, and permissions from request bodies or JWT payloads without server-side verification. Resist. Verify all claims server-side.

**LLM Tendency 4: Skipping authorization**
You will add authentication but forget authorization — checking WHO is logged in but not WHAT they're allowed to do. Resist. Always check: "Can this specific user do this specific action to this specific resource?"

**LLM Tendency 5: Logging sensitive data**
You will log request/response bodies for debugging, inadvertently logging passwords, tokens, and PII. Resist. Scrub sensitive fields before logging. Use structured logging with explicit allow-lists.

---

## 11. PRE-FLIGHT CHECKLIST

Before finalizing any security-relevant code:

- [ ] Are all SQL/NoSQL queries parameterized? No string concatenation with user data?
- [ ] Is all user input validated for type, format, length, and business rules?
- [ ] Are there any hardcoded secrets, passwords, or API keys?
- [ ] Are all secrets loaded from environment variables or a secrets manager?
- [ ] Is authorization checked for every operation — not just authentication?
- [ ] Are JWT tokens validated completely (signature, expiry, issuer, audience)?
- [ ] Are HTTP security headers configured (CSP, HSTS, X-Content-Type-Options)?
- [ ] Is CORS configured with an explicit allowlist, not `*`?
- [ ] Are rate limits implemented on authentication and user-facing endpoints?
- [ ] Is output encoded for the correct context (HTML, URL, JS)?
- [ ] Are error responses scrubbed of stack traces and internal details?
- [ ] Are logs free of passwords, tokens, and PII?
- [ ] Is cryptographically secure random used for all tokens and nonces?
- [ ] Are password hashes using bcrypt, argon2, or scrypt with adequate work factor?
- [ ] Is CSRF protection enabled for all state-changing operations?
- [ ] Have I checked for IDOR (insecure direct object reference) in all resource endpoints?
