# Routing Audit
**Phase G.11 — Domain & Routing Verification**
Generated: 2026-06-03
Method: HTTP probes (curl), SSL inspection, HTTP header analysis

---

## Domain Status

| Domain | DNS | HTTP | SSL | Server | Type | Result |
|--------|-----|------|-----|--------|------|--------|
| profiles.rald.cloud | ✅ | 200 | ✅ Google WE1, exp Aug 28 2026 | cloudflare | Cloudflare Pages (HTML SPA) | ✅ LIVE |
| auth.rald.cloud | ✅ | 200 | ✅ Let's Encrypt E8, exp Aug 22 2026 | cloudflare | Cloudflare Worker (JSON API) | ✅ LIVE |
| loop.rald.cloud | ✅ | 200 | ✅ Google WE1, exp Aug 25 2026 | cloudflare | Cloudflare Pages (HTML SPA) | ✅ LIVE |
| messenger.rald.cloud | ✅ | 401 | ✅ Google WE1, exp Aug 23 2026 | cloudflare | Cloudflare Worker (JSON API) | ⚠️ LIVE (API only) |
| accounts.rald.cloud | ✅ | 403 | ✅ | cloudflare | Cloudflare challenge | ❌ DEAD (deprecated) |
| loop-api.rald.cloud | ✅ | 404 | ✅ | cloudflare | Cloudflare Worker (JSON API) | ✅ LIVE |
| notification.rald.cloud | ❌ | 000 | N/A | N/A | NONE | ❌ NOT DEPLOYED |
| search.rald.cloud | ❌ | 000 | N/A | N/A | NONE | ❌ NOT DEPLOYED |
| crm.rald.cloud | ❌ | 000 | N/A | N/A | NONE | ❌ NOT DEPLOYED |
| inbox.rald.cloud | ❌ | 000 | N/A | N/A | NONE | ❌ NOT DEPLOYED |

---

## Routing Configuration Evidence

### profiles.rald.cloud
- **Type**: Cloudflare Pages
- **Serves**: React SPA (rald-auth-ui)
- **JS bundle**: `/assets/index-CDfGnSmz.js`
- **Auth URL in bundle**: `https://auth.rald.cloud` ✅ CORRECT

### auth.rald.cloud
- **Type**: Cloudflare Worker (`rald-auth` worker)
- **Route config** (wrangler.toml): `pattern = "auth.rald.cloud/*"`
- **Root response**: `{"docs":"https://auth.rald.cloud/health","service":"rald-auth","version":"2.1.0",...}`
- **All health endpoints working**: `/health`, `/healthz`, `/version`, `/ready`, `/system/status`, `/system/dependencies`

### loop.rald.cloud
- **Type**: Cloudflare Pages
- **Serves**: React SPA (Loop frontend)
- **JS bundle**: `/assets/index-C99bdGQF.js`
- **Auth URL in bundle** (BEFORE FIX): `https://accounts.rald.cloud?redirect_to=https://loop.rald.cloud/login&app_id=loop` ❌
- **Auth URL** (AFTER FIX, commit e6666a6): `https://profiles.rald.cloud` ✅
- **API URL in bundle**: `https://loop-api.rald.cloud` ✅ CORRECT
- **Meta description**: "Loop — built on Replit. Update this description to reflect the app." ⚠️ Stale default template description — cosmetic issue only

### loop-api.rald.cloud
- **Type**: Cloudflare Worker (`loop-api` worker)
- **Route config** (wrangler.toml): `pattern = "loop-api.rald.cloud/*"`
- **CORS origin** (production): `https://loop.rald.cloud,https://loop.ostinato-loop.pages.dev`
- **Working routes**: `/api/health`, `/api/auth/rald-sso`, `/api/auth/send-otp`, `/api/trending`, `/api/rooms`
- **Missing routes** (404): `/api/auth/me`, `/api/users`, `/api/conversations`, `/api/feed`, `/api/posts`

### messenger.rald.cloud
- **Type**: Cloudflare Worker (`loop-messenger-api` worker)
- **Route config** (wrangler.toml): `pattern = "messenger.rald.cloud/*"`
- **Root response**: 401 (auth middleware on all routes except /health)
- **Purpose**: Backend API only — conversations, messages, calls, users
- **What's missing**: Frontend SPA (not deployed to this domain)

### accounts.rald.cloud
- **Type**: Cloudflare (DNS resolves, but Cloudflare challenge activated)
- **Status**: 403 + JavaScript challenge page
- **Impact**: Every user clicking "Login" from Loop or Messenger is sent here — completely stuck

---

## Architecture Gap

| User Journey Step | Expected Domain | Actual Status |
|------------------|----------------|---------------|
| Sign up / Sign in | profiles.rald.cloud | ✅ Working |
| "Enter Loop" | loop.rald.cloud → redirects to profiles | ⚠️ Fixed (was accounts.rald.cloud) |
| "Enter Messenger" | messenger.rald.cloud | ❌ No frontend exists here |
| Notifications | notification.rald.cloud | ❌ Not deployed |
| Search | search.rald.cloud | ❌ Not deployed |
| CRM | crm.rald.cloud | ❌ Not deployed |
| Inbox | inbox.rald.cloud | ❌ Not deployed |
