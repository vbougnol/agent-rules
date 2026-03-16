---
trigger: model_decision
scope: security_review
---

# Security Audit Methodology

> Systematic security review framework based on OWASP guidelines and security best practices.
> Use this rule when reviewing code for security vulnerabilities or performing security audits.

---

## 1. Attack Surface Analysis

Before reviewing code, map the attack surface:

1. **Entry points** — APIs, forms, file uploads, webhooks, WebSocket endpoints
2. **Data flows** — trace data from input to storage to output
3. **Trust boundaries** — where does trusted/untrusted data cross?
4. **External dependencies** — list all dependencies and their versions
5. **Privileged operations** — identify admin, destructive, or sensitive actions

---

## 2. OWASP Top 10 Review Checklist

### 2.1 Injection (SQL, NoSQL, Command, LDAP)
- [ ] All queries are parameterized
- [ ] User input is never concatenated into queries
- [ ] ORM queries are safe from injection
- [ ] Shell command execution is avoided with user input
- [ ] Template rendering escapes user data

### 2.2 Broken Authentication
- [ ] Passwords hashed with strong algorithms (bcrypt, Argon2)
- [ ] MFA available for sensitive operations
- [ ] Session tokens are secure (HttpOnly, Secure, SameSite)
- [ ] Account lockout after failed attempts
- [ ] Token expiration is enforced

### 2.3 Sensitive Data Exposure
- [ ] Sensitive data encrypted at rest and in transit
- [ ] API keys and secrets in environment variables (not code)
- [ ] PII properly protected
- [ ] Error messages are generic (no stack traces in production)
- [ ] Logs do not contain secrets

### 2.4 XML External Entities (XXE)
- [ ] XML parsing disables external entities
- [ ] Safer data formats (JSON) used when possible

### 2.5 Broken Access Control
- [ ] All endpoints properly authorized
- [ ] IDOR (Insecure Direct Object Reference) protection in place
- [ ] CORS policies properly configured
- [ ] Principle of least privilege followed
- [ ] Tenant isolation enforced

### 2.6 Security Misconfiguration
- [ ] Default credentials changed
- [ ] Unnecessary features disabled
- [ ] Security headers set (CSP, X-Frame-Options, etc.)
- [ ] HTTPS enforced
- [ ] Debug mode disabled in production

### 2.7 Cross-Site Scripting (XSS)
- [ ] All user input escaped before rendering
- [ ] Content Security Policy in place
- [ ] Dangerous functions (`innerHTML`, `eval`) avoided
- [ ] Input validated on both client and server

### 2.8 Insecure Deserialization
- [ ] Untrusted data is never directly deserialized
- [ ] Safe alternatives used (JSON instead of pickle, etc.)

### 2.9 Components with Known Vulnerabilities
- [ ] Dependencies are up to date
- [ ] Process exists for security updates
- [ ] Vulnerability scanners in CI/CD (`npm audit`, Snyk, etc.)

### 2.10 Insufficient Logging & Monitoring
- [ ] Security events are logged
- [ ] Logs protected from tampering
- [ ] Alerting exists for suspicious activity
- [ ] Request IDs enable tracing

---

## 3. Security Headers Checklist

Verify these headers are set on all responses:

- [ ] `Strict-Transport-Security` (HSTS)
- [ ] `Content-Security-Policy` (CSP)
- [ ] `X-Content-Type-Options: nosniff`
- [ ] `X-Frame-Options: DENY`
- [ ] `X-XSS-Protection: 1; mode=block`
- [ ] `Referrer-Policy: no-referrer`
- [ ] `Permissions-Policy`

---

## 4. Authentication & Authorization Rules

- **AuthN is not AuthZ** — authentication (who you are) and authorization (what you can do) must be separate and explicit
- **Deny by default** — when unclear, deny access
- **Fail closed** — sensitive operations must fail safely
- **Sanitized errors** — clients get generic errors; detailed context goes to logs (without secrets)
- **Admin routes** — under explicit path prefix (e.g., `/api/admin/*`) with RBAC enforcement
- **Ordinary users** must receive `403` for admin-only behavior
- **Audit logs** — non-read admin actions must generate audit logs

---

## 5. Internal Endpoints

- `/internal/*` endpoints are hidden/protected by default
- Must not be exposed publicly by accident
- Protected by internal token or network policy
- Documented in ADR/runbooks

---

## 6. Risk Assessment Framework

For each vulnerability found, evaluate:

| Factor | Question |
|--------|----------|
| **Severity** | Critical / High / Medium / Low |
| **Likelihood** | How easy is it to exploit? |
| **Impact** | What's the damage if exploited? |
| **Priority** | Severity x Likelihood |

---

## 7. Vulnerability Report Format

When reporting a security issue, use this structure:

```
**[SEVERITY] Vulnerability Title**

- **Location**: File path, line number, or endpoint
- **Description**: What is the vulnerability?
- **Impact**: What can an attacker do?
- **Reproduction**: Steps to exploit
- **Remediation**: How to fix it
- **References**: CWE ID, OWASP category
```

---

## 8. Remediation Priorities

1. **Critical** — fix immediately, may require hotfix
2. **High** — fix in current sprint/iteration
3. **Medium** — fix in next sprint/iteration
4. **Low** — add to backlog, fix when convenient

Always provide:
- Specific fix recommendations with code examples
- References to security standards (OWASP, CWE)
- Defense-in-depth suggestions (multiple layers of protection)
