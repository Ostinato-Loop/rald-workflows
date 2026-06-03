# RALD SSO ARCHITECTURE V1
**Phase:** G.12 — Workstream 2
**Generated:** 2026-06-03 12:11 UTC

---

## CANONICAL FLOW

```
User → App (loop/messenger) → not authenticated
     → redirect: profiles.rald.cloud/login?app_id=loop&return_url=...
     → authenticate (OTP)
     → profiles.rald.cloud → auth.rald.cloud/sso/exchange
     → rald_token issued
     → redirect back to app/auth/callback?rald_token=XXX
     → App calls: POST app-api.rald.cloud/api/auth/rald-sso {rald_token}
     → Worker validates rald_token → issues app-scoped JWT
     → User enters app (no second login ever)
```

## TOKEN HIERARCHY

| Token | Issuer | TTL | Scope |
|-------|--------|-----|-------|
| RALD_JWT | auth.rald.cloud | 24h | Platform-wide |
| LOOP_JWT | loop-api.rald.cloud | 24h | Loop only |
| MESSENGER_JWT | messenger | 24h | Messenger only |

## IMPLEMENTATION STATUS

| Component | Status |
|-----------|--------|
| Auth API SSO endpoint | ✅ Working |
| profiles.rald.cloud redirect | ✅ Working |
| Loop SSO callback | ✅ Working |
| Messenger SSO callback | ⚠ Frontend not deployed |
| One-time token | ❌ Not implemented |
| App whitelist | ❌ Not implemented |

## SECURITY GAPS TO FIX (before Level 2)
1. rald_token must be invalidated after first use (replay attack prevention)
2. return_url must be validated against allowlist (open redirect prevention)
3. app_id must be validated against registered apps whitelist
