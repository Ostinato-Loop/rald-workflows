# SECURITY REVALIDATION REPORT
**Phase:** G.12 — Workstream 5
**Generated:** 2026-06-03 12:11 UTC
**Scope:** RALD Ecosystem — auth.rald.cloud, profiles.rald.cloud, loop-api.rald.cloud, messenger.rald.cloud

---

## EXECUTIVE SUMMARY

| Category | Finding | Status |
|----------|---------|--------|
| Default JWT secrets | None found | ✅ PASS |
| Committed .env files | None found | ✅ PASS |
| Exposed API keys in code | None found | ✅ PASS |
| OTP brute-force protection | Rate limiting present | ⚠ PARTIAL |
| OTP rate limiting | Implemented (Redis/KV) | ⚠ PARTIAL (KV not provisioned in all workers) |
| Session revocation | DB-backed revocation | ✅ PASS |
| Device management | Not implemented | ❌ GAP |
| Token expiration | 24h TTL enforced | ✅ PASS |
| Redirect validation | Open redirect possible | ❌ GAP |
| App whitelist validation | Not enforced | ❌ GAP |

**CRITICAL:** 0  
**HIGH:** 2  
**MEDIUM:** 3  
**LOW:** 2

---

## CRITICAL FINDINGS (0)

None.

---

## HIGH FINDINGS (2)

### H1 — Open Redirect in SSO Flow
**Location:** auth.tsx, profiles.rald.cloud redirect handler  
**Risk:** Attacker can construct a URL that, after authentication, redirects the user to an attacker-controlled domain  
**Evidence:** return_url parameter not validated against an allowlist  
**Fix Required:**
```typescript
const ALLOWED_RETURN_URLS = [
  'https://loop.rald.cloud',
  'https://messenger.rald.cloud',
  'https://pay.rald.cloud',
];
if (!ALLOWED_RETURN_URLS.some(u => return_url?.startsWith(u))) {
  return_url = 'https://profiles.rald.cloud'; // safe fallback
}
```

### H2 — SSO Token Not One-Time
**Location:** auth.rald.cloud /sso/exchange  
**Risk:** Intercepted rald_token can be replayed by attacker to hijack session  
**Evidence:** rald_token validated but not invalidated after first use  
**Fix Required:** Mark SSO tokens as used in Supabase after first exchange

---

## MEDIUM FINDINGS (3)

### M1 — OTP Rate Limiting Depends on Unprovisioned KV
**Location:** rald-notify, rald-search RATE_LIMIT_KV binding  
**Risk:** Rate limiting silently disabled if KV namespace not created in Cloudflare  
**Status:** wrangler.toml has placeholder KV ID

### M2 — JWT Secret Same Across Environments
**Location:** RALD_JWT_SECRET in GitHub Secrets  
**Risk:** Development usage of production secrets  
**Fix:** Use separate secrets per environment (staging/production)

### M3 — No CORS Precheck Validation on auth.rald.cloud /sso/exchange
**Location:** auth.rald.cloud  
**Risk:** CORS allows all RALD domains but doesn't validate the Origin header matches the requesting app_id  

---

## LOW FINDINGS (2)

### L1 — pnpm Audit Warnings
Packages with known non-critical advisories present in loop and messenger workspaces. Run `pnpm audit --fix` to update.

### L2 — HEARTBEAT_PATH Unused Variable
**Location:** messenger/heartbeat-button.tsx  
**Risk:** Dead code, not a security issue  

---

## VERIFIED SECURE

| Check | Method | Result |
|-------|--------|--------|
| JWT signing algorithm | Code review: HMAC-SHA256 | ✅ |
| JWT secret strength | Secrets manager: >32 chars random | ✅ |
| No .env files committed | Git tree audit | ✅ |
| No hardcoded secrets | Code grep | ✅ |
| OTP expiry enforced | Code review: 10min TTL | ✅ |
| HTTPS everywhere | DNS audit | ✅ |
| Session stored in DB | Supabase: sessions table | ✅ |
| SQL injection | Supabase parameterized queries | ✅ |

---

## TARGET STATUS

CRITICAL = 0 ✅  
HIGH = 2 ❌ (requires implementation work)

*Generated for RALD / LILCKY STUDIO LIMITED*
