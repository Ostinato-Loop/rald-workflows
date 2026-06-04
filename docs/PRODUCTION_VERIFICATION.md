# RALD Production Verification Report
> Task 6 — Foundation Stabilization Sprint | 2026-06-04

## CI / Deploy Matrix

| Service | CI | Deploy | Fail-Fast | Head |
|---------|----|----|-----------|------|
| rald-auth-core | ✅ | ✅ | ✅ | 4e9c7f2 |
| rald-auth-ui | ✅ | ✅ | N/A | 42478e9 |
| loop-api | ✅ | ✅ | ✅ | 0dff42f |
| messenger | ✅ | ✅ | ⚠️ | 0a71686 |
| rald-notify | ✅ | ✅ | ✅ | 54c911b |
| rald-search | ✅ | ✅ | ✅ | 14bdee9 |
| rald-inbox | ✅ | ✅ | ✅ | 412b51e |
| rald-realtime | ✅ | ✅ | ✅ | 4a673f4 |

## End-to-End Journey Verification

1. profiles.rald.cloud — ✅ SPA serves login form
2. POST auth.rald.cloud/auth/login — ✅ Returns JWT. Secrets deployed. Fail-fast active.
3. Redirect to loop.rald.cloud — ✅ RALD JWT validated by loop-api. Same RALD_JWT_SECRET.
4. Open messenger.rald.cloud — ✅ RALD JWT accepted. CI + Deploy green.
5. Stay authenticated — ✅ 7-day token. All API calls validated per-request.

## What Works
- ✅ Full login flow (password + SMS OTP + email OTP)
- ✅ Cross-app SSO (RALD JWT accepted across all 7 workers)
- ✅ All CI pipelines green
- ✅ All deploys green
- ✅ Fail-fast on missing secrets (6/7 workers — messenger pending)

## What Needs Action
| Issue | Priority |
|-------|----------|
| KV namespaces not provisioned (REPLACE_WITH_*) | HIGH |
| Messenger fail-fast not added | MEDIUM |
| RALD_SESSION_KV missing → logout non-functional | MEDIUM |
| No staging environment | MEDIUM |
| Cron triggers not registered in CF dashboard (notify, inbox) | LOW |

## Sprint Success Criteria

| Criteria | Status |
|----------|--------|
| Profiles login works | ✅ |
| Loop login works | ✅ |
| Messenger login works | ✅ |
| Cross-app SSO works | ✅ |
| Missing secrets → 503 | ✅ (6/7 workers) |
| Service dependency map | ✅ docs/SERVICE_DEPENDENCY_MAP.md |
| WIZMAC operational shell | ✅ admin.rald.cloud (rald-control-center) |
