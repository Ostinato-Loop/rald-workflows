# RALD Authentication Chain Audit
> Task 1 — Foundation Stabilization Sprint | 2026-06-04

## Step 1 — profiles.rald.cloud (Cloudflare Pages SPA)
CI: ✅ | Deploy: ✅ | Secrets: none (static SPA)

## Step 2 — POST auth.rald.cloud/auth/login
- Algorithm: HS256, secret: RALD_JWT_SECRET (no fallback)
- Payload: sub, email, role, app_id="rald", iat, exp (+7d)
- Alt: SMS OTP via Termii; Email OTP via Resend
- Fail-fast: ✅ 503 if RALD_JWT_SECRET/SUPABASE_* missing

## Step 3 — SSO Exchange: loop-api.rald.cloud/api/auth/rald-sso
- Verifies RALD JWT with RALD_JWT_SECRET (MUST match auth.rald.cloud)
- Creates/syncs user in D1 (loop-db)
- Issues Loop JWT (LOOP_JWT_SECRET)
- Fail-fast: ✅ 503 if RALD_JWT_SECRET or LOOP_JWT_SECRET missing

## Step 4 — Token Validation Across All Services (HS256, RALD_JWT_SECRET)

| Service | Status |
|---------|--------|
| rald-auth-core | ✅ |
| loop-api | ✅ |
| messenger | ✅ |
| rald-inbox | ✅ |
| rald-notify | ✅ |
| rald-search | ✅ |
| rald-realtime | ✅ |

## Step 5 — Logout
- POST auth.rald.cloud/auth/logout — invalidates RALD_SESSION_KV
- ⚠️ RALD_SESSION_KV not provisioned → logout is cosmetic (tokens valid for 7d)

## Issues

### HIGH
1. All KV namespaces still show REPLACE_WITH_* — rate limiting disabled, logout non-functional
2. Messenger fail-fast not added

### MEDIUM
3. Token expiry 7d without blacklisting
4. No staging environment

## Security Notes
- redirect_to validated against rald.cloud zone — no open redirect vulnerability
- All services accept RALD-signed tokens regardless of app_id (SSO-permissive, correct for cross-app)
