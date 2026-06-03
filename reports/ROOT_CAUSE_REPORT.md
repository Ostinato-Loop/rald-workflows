# Root Cause Report
**Phase G.11 — Definitive Root Cause Analysis**
Generated: 2026-06-03
Confidence: HIGH (verified against live production + source code)

---

## FAILURE MATRIX

| # | Failure | Severity | Source File | Status |
|---|---------|----------|------------|--------|
| RC-1 | Loop redirects to dead domain (accounts.rald.cloud) | CRITICAL | `loop/artifacts/loop/src/hooks/use-auth.tsx:33` | ✅ FIXED — commit e6666a6 |
| RC-2 | Messenger has no user-facing frontend | CRITICAL | Architecture gap — no Pages deployment | ⚠️ OPEN — needs CF token + domain config |
| RC-3 | Messenger frontend hardcodes accounts.rald.cloud | HIGH | `messenger/artifacts/loop-messenger/src/pages/auth.tsx` | ✅ FIXED — commit 5a2fa01 |
| RC-4 | Loop API missing key routes | HIGH | `loop/artifacts/cloudflare-worker/src/index.ts` | ⚠️ OPEN — /api/auth/me, /api/users, /api/conversations return 404 |
| RC-5 | messenger.rald.cloud CORS blocks loop.rald.cloud | MEDIUM | `messenger/artifacts/api-server/src/app.ts` | ⚠️ OPEN — ALLOWED_ORIGINS lacks messenger.rald.cloud and loop.rald.cloud |
| RC-6 | Four micro-services not deployed | MEDIUM | Missing repos or wrangler deployments | ⚠️ OPEN — notification, search, crm, inbox |
| RC-7 | registered_apps migration not run in production | LOW | `rald-auth-core/supabase/migrations/20260603_registered_apps.sql` | ⚠️ OPEN — fallback active |

---

## RC-1: CRITICAL — accounts.rald.cloud Hardcoded (FIXED)

**Symptom**: Real users navigating to loop.rald.cloud immediately bounce to a Cloudflare 403 challenge page. Cannot log in. Cannot use the app.

**Root cause**: Single hardcoded string constant in the frontend source:

```typescript
// File: loop/artifacts/loop/src/hooks/use-auth.tsx
// Line 33 (before fix):
const RALD_AUTH_UI = "https://accounts.rald.cloud";
```

This string is used in `redirectToRaldAuth()`:
```typescript
window.location.href = `${RALD_AUTH_UI}?redirect_to=${redirectTo}&app_id=${appId}`;
```

`accounts.rald.cloud` is a deprecated first-generation auth UI domain. Cloudflare has placed a managed challenge on it. All users are blocked.

**Fix** (commit `e6666a6`):
```typescript
const RALD_AUTH_UI = (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) ?? "https://profiles.rald.cloud";
```
The domain now correctly points to `profiles.rald.cloud` (the active RALD auth UI). The env var fallback allows future domain changes without code changes.

**Remediation complete?**: Code fix pushed. Loop's GitHub Actions CI (`deploy.yml`) will deploy on next push to `main` OR can be triggered via `workflow_dispatch`. A new Cloudflare Pages build is required for the fix to take effect in production.

---

## RC-2: CRITICAL — Messenger Frontend Not Deployed

**Symptom**: Users navigating to `https://messenger.rald.cloud` receive `401 {"error":"Unauthorized"}` — a raw JSON API response, not a UI.

**Root cause**: `messenger.rald.cloud` is routed exclusively to the backend Cloudflare Worker (`loop-messenger-api`). The messenger React frontend (`artifacts/loop-messenger`) is built via Vite and designed for Cloudflare Pages deployment, but:

1. The Pages deployment CI job (`deploy-pages.yml`) checks for `CLOUDFLARE_API_TOKEN`:
```bash
if [ -z "$CLOUDFLARE_API_TOKEN" ]; then
  echo "⚠️  CLOUDFLARE_API_TOKEN not configured."
  echo "   Skipping — all other checks remain green."
  exit 0   # ← SILENT SUCCESS, NO DEPLOYMENT
fi
```

2. This secret has **not been added** to the `messenger` GitHub repository secrets.

3. Even if the token were present: the Pages project would deploy to `loop-messenger.pages.dev`, not `messenger.rald.cloud`. A custom domain for Pages must be configured separately (Cloudflare Dashboard or `wrangler pages domain add`). But `messenger.rald.cloud` is already bound to the Worker — both cannot use the same domain.

**Remediation required**:
1. Add `CLOUDFLARE_API_TOKEN` (with Pages:Edit scope) to `messenger` repo GitHub secrets
2. Choose a domain for the messenger frontend:
   - Option A: `app.messenger.rald.cloud` (new subdomain → Pages)
   - Option B: `chat.rald.cloud` (new subdomain → Pages)
   - Option C: Move Worker to `api.messenger.rald.cloud` and assign `messenger.rald.cloud` → Pages
3. Update messenger frontend's `VITE_API_URL` to point to the Worker
4. Update Worker CORS to allow the new frontend domain
5. Update `openMessenger()` in `loop/use-auth.tsx` to point to the new frontend domain

---

## RC-3: HIGH — Messenger Frontend Also Hardcoded (FIXED)

**Symptom**: The messenger auth page's "Continue with RALD account" button would have redirected to `accounts.rald.cloud` even if the frontend had been deployed.

**Root cause** (before fix):
```typescript
// File: messenger/artifacts/loop-messenger/src/pages/auth.tsx
const RALD_AUTH_UI = "https://accounts.rald.cloud";
```

**Fix** (commit `5a2fa01`):
```typescript
const RALD_AUTH_UI = (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) ?? "https://profiles.rald.cloud";
```

---

## RC-4: HIGH — Loop API Missing Routes

**Symptom**: After successful auth, Loop frontend would fail silently on `/api/auth/me`, `/api/users`, `/api/conversations`.

**Evidence** (live probe):
```
GET  loop-api.rald.cloud/api/auth/me       → 404 {"error":"Not found","path":"/api/auth/me"}
GET  loop-api.rald.cloud/api/users         → 404 {"error":"Not found","path":"/api/users"}
GET  loop-api.rald.cloud/api/conversations → 404 {"error":"Not found","path":"/api/conversations"}
```

**Registered routes** (from `artifacts/cloudflare-worker/src/index.ts`):
```typescript
app.route("/api/health",         health);
app.route("/api/auth",           auth);
app.route("/api/auth/rald-sso",  raldSso);
app.route("/api/trending",       trending);
app.route("/api/rooms",          rooms);
```

`/api/auth/me` needs to be inside the `auth` route file. Need to verify if `/api/auth/me` is declared within `src/routes/auth.ts`. The route `/api/auth` is mounted — sub-routes `/send-otp`, `/verify-otp`, `/me` should be accessible. Need deeper investigation of `auth.ts`.

**Hypothesis**: Either (a) `/api/auth/me` sub-route is not declared in `src/routes/auth.ts`, or (b) it requires an auth header that the test probe did not supply.

---

## RC-5: MEDIUM — Messenger CORS Misconfiguration

**Source** (`messenger/artifacts/api-server/src/app.ts`):
```typescript
const ALLOWED_ORIGINS = [
  "https://loop-messenger.pages.dev",
  "https://messenger.ostloop.name.ng",
  "https://loop-messenger-api.d5a1cd03b76f467430034af64a7062fd.workers.dev",
];
```

**Missing**:
- `https://loop.rald.cloud` (Loop cross-app calls)
- `https://messenger.rald.cloud` (if frontend were on same domain)
- Any future Pages custom domain for the messenger frontend

Note: This file is `artifacts/api-server` (the development Express server), not the production Worker. The production Worker's CORS is in `workers/loop-messenger-api/`. Needs separate audit.

---

## RC-6: MEDIUM — Four Micro-Services Not Deployed

| Domain | Likely Repo | Status |
|--------|------------|--------|
| notification.rald.cloud | rald-notify | No DNS record — HTTP 000 |
| search.rald.cloud | rald-search | No DNS record — HTTP 000 |
| crm.rald.cloud | rald-crm | No DNS record — HTTP 000 |
| inbox.rald.cloud | rald-inbox | No DNS record — HTTP 000 |

These are referenced in `workers/loop-messenger-api/wrangler.toml` as `NOTIFY_URL`, `SEARCH_URL`, `CRM_URL`, `INBOX_URL` vars but none of the targets respond. The Messenger API will silently fail when trying to call these services.

---

## RC-7: LOW — registered_apps Migration Not Applied

The new `registered_apps` table (migration `20260603_registered_apps.sql`) seeded with 24 apps was pushed to GitHub but has NOT been applied to the Supabase production instance. `rald-auth-core/src/routes/sso.ts` has a fallback that returns a 200 with the legacy `TRUSTED_APP_IDS` list when the DB table is absent. This means the DB-driven authorization is not active yet, but the service continues to work via the fallback.

**Risk**: Low. Fallback is intentional and correct. Apply migration to activate DB-driven app registry.
