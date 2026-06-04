# RALD Service Dependency Map
> Generated: 2026-06-04 | Sprint: Foundation Stabilization
> Machine-readable dependency graph for all production RALD services.

---

## Identity Layer

### auth.rald.cloud (rald-auth-core)
- **Worker**: Cloudflare Worker — `rald-auth`
- **Route**: auth.rald.cloud/*
- **DB**: Supabase PostgreSQL (onxdcikfttdwnhofsuwo.supabase.co)
  - Tables: users, sessions, otp_codes, audit_logs, devices, profiles
- **KV**:
  - RATE_LIMIT_KV — rald-auth-rate-limit (REPLACE_WITH_RATE_LIMIT_KV_NAMESPACE_ID)
  - RALD_SESSION_KV — rald-session (REPLACE_WITH_RALD_SESSION_KV_NAMESPACE_ID)
- **External**:
  - Termii (SMS OTP)
  - Resend (Email OTP / Welcome emails)
  - Clerk (legacy SSO bridge — CLERK_SECRET_KEY, CLERK_PUBLISHABLE_KEY)
- **Required Secrets**: RALD_JWT_SECRET, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, TERMII_API_KEY, TERMII_SENDER_ID, RESEND_API_KEY, CLERK_SECRET_KEY, CLERK_PUBLISHABLE_KEY
- **Exposes**: JWT tokens (HS256, RALD_JWT_SECRET), SSO exchange endpoint (/sso/exchange), Session broker (/session)
- **Fail-fast**: ✅ (added 2026-06-04)

### profiles.rald.cloud (rald-auth-ui)
- **Deployment**: Cloudflare Pages — `rald-auth-ui`
- **Type**: React SPA (Vite, TypeScript)
- **Depends on**: auth.rald.cloud (all auth calls)
- **Env vars** (build-time): VITE_AUTH_API_URL=https://auth.rald.cloud, VITE_APP_NAME=RALD
- **No runtime secrets** (static SPA)

---

## Community Layer

### loop-api.rald.cloud (loop — worker)
- **Worker**: Cloudflare Worker — `loop-api`
- **Route**: loop-api.rald.cloud/*
- **DB**:
  - Cloudflare D1 — `loop-db` (4616fcac-96e0-4150-a42f-3d020f45cd1d) — primary SQL
  - Supabase PostgreSQL — supplemental (profiles sync)
- **KV**:
  - CACHE (3c71da01b3174d6c9353adbfde7491a3) — sessions, rate limits, OTP store
- **R2**: loop-media — audio, images, attachments
- **Queues**: loop-tasks (producer + consumer, batch 10, timeout 5s)
- **Durable Objects**: RoomSession (sqlite class, migration v1)
- **AI**: Workers AI binding
- **Auth**: Validates RALD JWT (from auth.rald.cloud) via /api/auth/rald-sso
- **External**: Termii (SMS OTP for loop auth)
- **Fail-fast**: ⚠️ Pending (loop worker has no explicit fail-fast yet)

### loop.rald.cloud (loop — frontend)
- **Deployment**: Cloudflare Pages
- **Type**: React SPA (Vite, pnpm monorepo)
- **Depends on**: loop-api.rald.cloud, auth.rald.cloud

---

## Relationship Layer

### messenger.rald.cloud (messenger — API worker)
- **Worker**: Cloudflare Worker (pnpm monorepo, artifacts/worker)
- **Route**: messenger.rald.cloud/* (API)
- **Auth**: Validates RALD JWT (RALD_JWT_SECRET)
- **DB**: Supabase PostgreSQL
- **Real-time**: rald-realtime abstraction layer
- **Deployment**: Cloudflare Pages (messenger frontend — no custom domain yet)
- **Fail-fast**: ⚠️ Pending

---

## Platform Services

### inbox.rald.cloud (rald-inbox)
- **Worker**: Cloudflare Worker — `rald-inbox`
- **Route**: inbox.rald.cloud/*
- **DB**: Supabase PostgreSQL
  - Tables: conversations, messages, assignments, tags, views, sla_configs, audit_logs
- **KV**: RATE_LIMIT_KV (REPLACE_WITH_RATE_LIMIT_KV_ID)
- **Cron**: Every 10 minutes — SLA alert processing
- **Calls** (HTTP): notification.rald.cloud (NOTIFICATION_SERVICE_URL), search.rald.cloud (SEARCH_SERVICE_URL)
- **Auth**: RALD JWT validation
- **Required Secrets**: RALD_JWT_SECRET, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY
- **Fail-fast**: ✅ (added 2026-06-04)

### notification.rald.cloud (rald-notify)
- **Worker**: Cloudflare Worker — `rald-notify`
- **Route**: notification.rald.cloud/*
- **DB**: Supabase PostgreSQL
  - Tables: notifications, notification_channels, notification_deliveries, notification_templates, notification_preferences, notification_events, audit_logs
- **KV**: RATE_LIMIT_KV (REPLACE_WITH_RATE_LIMIT_KV_ID)
- **Cron**: Every 5 minutes — retry queue + scheduled delivery (manual CF dashboard setup required)
- **External**:
  - Resend (email delivery)
  - Termii (SMS delivery) — optional
  - Twilio (SMS fallback) — optional
  - VAPID (Web Push) — optional
- **Required Secrets**: RALD_JWT_SECRET, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, RESEND_API_KEY
- **Fail-fast**: ✅ (added 2026-06-04)

### search.rald.cloud (rald-search)
- **Worker**: Cloudflare Worker — `rald-search`
- **Route**: search.rald.cloud/*
- **DB**: Supabase PostgreSQL
  - Tables: entity tables (customers, users, workspaces, etc.), search_recent, search_saved, search_audit
- **KV**: RATE_LIMIT_KV (REPLACE_WITH_RATE_LIMIT_KV_ID)
- **Search Providers** (pluggable):
  - postgres (default) — full-text search via tsvector
  - meilisearch — optional (MEILISEARCH_HOST, MEILISEARCH_API_KEY)
  - opensearch — optional (OPENSEARCH_HOST, OPENSEARCH_API_KEY)
- **Required Secrets**: RALD_JWT_SECRET, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY
- **Fail-fast**: ✅ (added 2026-06-04)

### realtime.rald.cloud (rald-realtime)
- **Worker**: Cloudflare Worker — `rald-realtime`
- **Route**: realtime.rald.cloud/*
- **KV**:
  - RATE_LIMIT_KV (REPLACE_WITH_RATE_LIMIT_KV_ID)
  - HEALTH_KV (REPLACE_WITH_HEALTH_KV_ID)
  - PROVIDER_STATE_KV (REPLACE_WITH_PROVIDER_STATE_KV_ID)
- **Providers** (failover chain):
  1. Cloudflare RealtimeKit (CALLS_APP_ID, CALLS_APP_SECRET) — primary
  2. LiveKit (LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET) — fallback P2
  3. Tencent TRTC (TENCENT_SDK_APP_ID, TENCENT_SECRET_KEY) — fallback P3
- **Required Secrets**: at least one of CALLS_APP_SECRET, LIVEKIT_API_SECRET, TENCENT_SECRET_KEY
- **Fail-fast**: ✅ (added 2026-06-04)

---

## Shared Authentication Flow

```
User Browser
  └─ GET profiles.rald.cloud (Cloudflare Pages SPA)
       └─ POST auth.rald.cloud/auth/login  (password/OTP)
            └─ SUPABASE_URL (PostgreSQL: users table)
            └─ TERMII / RESEND (OTP delivery)
            └─ Returns: { token: JWT(RALD_JWT_SECRET) }
  └─ Redirect to loop.rald.cloud?token=...
       └─ GET loop-api.rald.cloud/api/auth/rald-sso
            └─ Validates RALD JWT with RALD_JWT_SECRET (MUST MATCH auth.rald.cloud)
            └─ Returns: Loop session token
  └─ Open messenger.rald.cloud
       └─ Validates RALD JWT with RALD_JWT_SECRET (MUST MATCH)
```

**Critical**: RALD_JWT_SECRET MUST be identical across:
rald-auth-core, loop-api, messenger, rald-inbox, rald-notify, rald-search, rald-realtime

---

## KV Namespace Registry

| Namespace | Binding | Used By | Status |
|-----------|---------|---------|--------|
| rald-auth-rate-limit | RATE_LIMIT_KV | rald-auth-core | ⚠️ REPLACE_WITH_* |
| rald-session | RALD_SESSION_KV | rald-auth-core | ⚠️ REPLACE_WITH_* |
| rald-notify-kv | RATE_LIMIT_KV | rald-notify | ⚠️ REPLACE_WITH_* |
| rald-search-kv | RATE_LIMIT_KV | rald-search | ⚠️ REPLACE_WITH_* |
| rald-inbox-kv | RATE_LIMIT_KV | rald-inbox | ⚠️ REPLACE_WITH_* |
| realtime-rate-limit | RATE_LIMIT_KV | rald-realtime | ⚠️ REPLACE_WITH_* |
| realtime-health | HEALTH_KV | rald-realtime | ⚠️ REPLACE_WITH_* |
| realtime-state | PROVIDER_STATE_KV | rald-realtime | ⚠️ REPLACE_WITH_* |
| loop-cache | CACHE | loop-api | ✅ 3c71da01b3174d6c9353adbfde7491a3 |

---

## External Service Dependencies

| Service | Used By | Purpose | Required |
|---------|---------|---------|----------|
| Supabase | All workers | PostgreSQL database | ✅ Critical |
| Termii | rald-auth-core, loop-api | SMS OTP delivery | ✅ Critical |
| Resend | rald-auth-core, rald-notify | Email delivery | ✅ Critical |
| Clerk | rald-auth-core | Legacy SSO bridge | ⚠️ Legacy |
| Cloudflare RealtimeKit | rald-realtime | Audio/video P1 | Conditional |
| LiveKit | rald-realtime | Audio/video P2 | Conditional |
| Tencent TRTC | rald-realtime | Audio/video P3 | Conditional |
| Meilisearch | rald-search | Search P2 | Optional |
| OpenSearch | rald-search | Search P3 | Optional |

---

## Deployment Status Matrix

| Service | Domain | CI | Deploy | Fail-Fast |
|---------|--------|----|--------|-----------|
| rald-auth-core | auth.rald.cloud | ✅ | ✅ | ✅ |
| rald-auth-ui | profiles.rald.cloud | ✅* | ✅ | N/A (SPA) |
| loop-api | loop-api.rald.cloud | ✅ | ✅ | ⚠️ |
| messenger | messenger.rald.cloud | ✅ | ✅ | ⚠️ |
| rald-notify | notification.rald.cloud | ✅* | ✅ | ✅ |
| rald-search | search.rald.cloud | ✅* | ✅ | ✅ |
| rald-inbox | inbox.rald.cloud | ✅ | ✅ | ✅ |
| rald-realtime | realtime.rald.cloud | ✅ | ✅ | ✅ |

*CI fix in progress (2026-06-04 stabilization sprint)
