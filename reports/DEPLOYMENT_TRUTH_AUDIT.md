# Deployment Truth Audit
**Phase G.11 — Evidence-Based Deployment Verification**
Generated: 2026-06-03
Method: GitHub API (commits) + live HTTP health endpoints

---

## Key: GitHub vs Production

| Service | GitHub HEAD | Deployed Version | Match | Status |
|---------|------------|-----------------|-------|--------|
| rald-auth-core | dae00f2 (2026-06-03) | v2.1.0 (live) | UNVERIFIABLE* | ✅ LIVE |
| rald-auth-ui (profiles.rald.cloud) | ca28b11 (2026-06-03) | Unknown — Cloudflare Pages | UNVERIFIABLE* | ✅ LIVE |
| loop (loop.rald.cloud) | e6666a6 (2026-06-03) | Unknown — Cloudflare Pages | UNVERIFIABLE* | ⚠️ LIVE (auth broken — now fixed) |
| loop-api (loop-api.rald.cloud) | e2135c4 (2026-06-03) | v1.0.0 (live) | UNVERIFIABLE* | ✅ LIVE (partial routes) |
| messenger API (messenger.rald.cloud) | 7ed8528 (2026-06-03) | v1.0.0 (live) | UNVERIFIABLE* | ✅ LIVE (API only) |
| messenger frontend | 5a2fa01 (2026-06-03) | NOT DEPLOYED | FAIL | ❌ NO FRONTEND |
| rald-notify | 27d23d0 (2026-06-03) | — | FAIL | ❌ NOT DEPLOYED |
| rald-search | c70f2a5 (2026-06-03) | — | FAIL | ❌ NOT DEPLOYED |
| notification.rald.cloud | — | — | FAIL | ❌ NO DNS |
| search.rald.cloud | — | — | FAIL | ❌ NO DNS |
| crm.rald.cloud | — | — | FAIL | ❌ NO DNS |
| inbox.rald.cloud | — | — | FAIL | ❌ NO DNS |

*UNVERIFIABLE: Cloudflare Workers and Pages do not expose deployed git SHA via health endpoint. Version strings (v1.0.0, v2.1.0) are hardcoded in source, not derived from commit SHA. The only way to confirm matching would be via Cloudflare API with a CF_API_TOKEN, which is not available in this audit environment.

---

## Live Service Evidence

### auth.rald.cloud
```
GET https://auth.rald.cloud/health
→ 200 {"status":"ok","service":"rald-auth","version":"2.1.0","environment":"production"}

GET https://auth.rald.cloud/ready
→ 200 {"ready":true,"checks":{"supabase":true,"jwt":true,"termii":true,"resend":true,"clerk":false,"rate_limit_kv":true,"session_kv":true}}
```

### loop-api.rald.cloud
```
GET https://loop-api.rald.cloud/api/health
→ 200 {"ok":true,"service":"loop-api","version":"1.0.0","environment":"production","bindings":{"db":true,"cache":true,...}}
```

### messenger.rald.cloud
```
GET https://messenger.rald.cloud/health
→ 200 {"status":"ok","service":"loop-messenger-api","version":"1.0.0","environment":"production"}
```

### SSL Certificates
| Domain | Issuer | Valid Until |
|--------|--------|------------|
| profiles.rald.cloud | Google Trust Services (WE1) | Aug 28 2026 |
| auth.rald.cloud | Let's Encrypt (E8) | Aug 22 2026 |
| loop.rald.cloud | Google Trust Services (WE1) | Aug 25 2026 |
| messenger.rald.cloud | Google Trust Services (WE1) | Aug 23 2026 |

All SSL certificates: ✅ VALID

---

## Deployment Gaps

| Domain | Issue |
|--------|-------|
| notification.rald.cloud | HTTP 000 (no DNS record, no deployment) |
| search.rald.cloud | HTTP 000 (no DNS record, no deployment) |
| crm.rald.cloud | HTTP 000 (no DNS record, no deployment) |
| inbox.rald.cloud | HTTP 000 (no DNS record, no deployment) |
| messenger frontend | Build exists in repo, Pages deploy skipped (CLOUDFLARE_API_TOKEN missing in repo secrets) |
