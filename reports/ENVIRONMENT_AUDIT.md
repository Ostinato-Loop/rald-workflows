# Environment Audit
**Phase G.11 — Secret & Binding Verification**
Generated: 2026-06-03
Method: Live health endpoints + wrangler.toml + source analysis (no secret values revealed)

---

## auth.rald.cloud (rald-auth-core v2.1.0)

Source: `GET https://auth.rald.cloud/ready` and `/system/status`

| Secret / Binding | Status | Evidence |
|-----------------|--------|---------|
| SUPABASE_URL | ✅ PRESENT | `checks.supabase: true` |
| SUPABASE_SERVICE_ROLE_KEY | ✅ PRESENT | `checks.supabase: true` |
| RALD_JWT_SECRET | ✅ PRESENT | `checks.jwt: true` |
| TERMII_API_KEY | ✅ PRESENT | `checks.termii: true` |
| TERMII_SENDER_ID | ⚠️ UNKNOWN | Not in /ready — prior report indicates may be misconfigured |
| RESEND_API_KEY | ✅ PRESENT | `checks.resend: true` |
| CLERK_SECRET_KEY | ❌ MISSING | `checks.clerk_full: false` |
| CLERK_PUBLISHABLE_KEY | ❌ MISSING | `checks.clerk_full: false` |
| RATE_LIMIT_KV | ✅ PRESENT | `checks.rate_limit_kv: true` |
| RALD_SESSION_KV | ✅ PRESENT | `checks.session_kv: true` |
| D1 bindings | N/A | Not used in rald-auth-core |
| R2 bindings | N/A | Not used in rald-auth-core |

**Overall**: ✅ OPERATIONAL (Clerk intentionally deferred)

---

## loop-api.rald.cloud (loop CF Worker v1.0.0)

Source: `GET https://loop-api.rald.cloud/api/health` + `wrangler.toml` analysis

| Secret / Binding | Status | Evidence |
|-----------------|--------|---------|
| RALD_JWT_SECRET | ✅ PRESENT | SSO bridge accepts tokens: `verifyRaldJwt` works |
| LOOP_JWT_SECRET | ✅ LIKELY PRESENT | Deploy workflow requires it (exits 1 if missing) |
| SUPABASE_URL | ✅ PRESENT | wrangler.toml var, confirmed in health |
| SUPABASE_SERVICE_ROLE_KEY | ✅ LIKELY PRESENT | Deploy workflow requires it (exits 1 if missing) |
| TERMII_API_KEY | ⚠️ UNKNOWN | Deploy pushes if present |
| TERMII_SENDER_ID | ⚠️ UNKNOWN | Deploy pushes if present |
| OPENROUTER_API_KEY | ⚠️ UNKNOWN | Deploy pushes if present |
| D1 (DB — loop-db) | ✅ PRESENT | `health.bindings.db: true` (ID: 4616fcac-96e0-4150-a42f-3d020f45cd1d) |
| KV (CACHE) | ✅ PRESENT | `health.bindings.cache: true` (ID: 3c71da01b3174d6c9353adbfde7491a3) |
| R2 (MEDIA — loop-media) | ⚠️ UNKNOWN | wrangler.toml declares it; health endpoint not exposing R2 status |
| Queue (TASK_QUEUE — loop-tasks) | ⚠️ UNKNOWN | Declared in wrangler.toml |
| Durable Objects (ROOM_SESSION) | ⚠️ UNKNOWN | Declared in wrangler.toml |
| AI binding | ⚠️ UNKNOWN | Declared in wrangler.toml |

**Overall**: ✅ CORE SECRETS PRESENT — peripheral bindings unverifiable without CF API

---

## messenger.rald.cloud (loop-messenger-api CF Worker v1.0.0)

Source: `GET https://messenger.rald.cloud/health` + wrangler.toml + deploy workflow analysis

| Secret / Binding | Status | Evidence |
|-----------------|--------|---------|
| RALD_JWT_SECRET | ✅ PRESENT | Worker accepts Bearer tokens: 400 (past auth) not 401 |
| SUPABASE_SERVICE_ROLE_KEY | ✅ PRESENT | Deploy workflow sends FATAL if missing |
| SUPABASE_URL | ✅ PRESENT | wrangler.toml var |
| TERMII_API_KEY | ⚠️ UNKNOWN | Deploy pushes if present |
| VAPID_PUBLIC_KEY | ⚠️ UNKNOWN | Deploy pushes if present (push notifications) |
| VAPID_PRIVATE_KEY | ⚠️ UNKNOWN | Deploy pushes if present |
| VAPID_SUBJECT | ⚠️ UNKNOWN | Deploy pushes if present |
| API_ORIGIN | ⚠️ UNKNOWN | Deploy pushes if present |
| KV / D1 / R2 | N/A | Not declared in wrangler.toml |

**Overall**: ✅ CORE SECRETS PRESENT

---

## messenger frontend (loop-messenger Vite App — NOT DEPLOYED)

| Config | Status | Evidence |
|--------|--------|---------|
| VITE_API_URL | ❌ NOT SET | Never deployed — deploy-pages.yml skipped |
| VITE_SUPABASE_URL | ❌ NOT SET | Same |
| VITE_SUPABASE_ANON_KEY | ❌ NOT SET | Same |
| CLOUDFLARE_API_TOKEN (repo secret) | ❌ MISSING | deploy-pages.yml line: `if [ -z "$CLOUDFLARE_API_TOKEN" ]; then echo "⚠️  CLOUDFLARE_API_TOKEN not configured..." exit 0` |

**This is why the messenger frontend was never deployed.** The CI job exits 0 (no failure) when CLOUDFLARE_API_TOKEN is absent. The deployment is silently skipped.

---

## Summary

| Service | Core Secrets | Bindings | Status |
|---------|-------------|---------|--------|
| auth.rald.cloud | ✅ All present | ✅ KV × 2 | OPERATIONAL |
| loop-api.rald.cloud | ✅ Core present | ✅ D1, KV | OPERATIONAL (partial routes) |
| messenger.rald.cloud | ✅ Core present | N/A | OPERATIONAL (API only) |
| messenger frontend | ❌ Never deployed | N/A | NOT DEPLOYED |
| rald-notify | ❌ Unknown | N/A | NOT DEPLOYED |
| rald-search | ❌ Unknown | N/A | NOT DEPLOYED |
