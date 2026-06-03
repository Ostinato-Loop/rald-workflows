# MESSENGER LOGIN ROOT CAUSE
**Phase:** G.12 — Workstream 4  
**Generated:** 2026-06-03 12:06 UTC

---

## EXECUTIVE SUMMARY

Messenger has two separate root causes:
1. **Auth URL hardcoded** to dead domain (same as Loop) — FIXED
2. **Frontend SPA never deployed** — messenger.rald.cloud serves only the API Worker

---

## ROOT CAUSE 1: Hardcoded Auth URL

**File:** `artifacts/loop-messenger/src/pages/auth.tsx`  

```typescript
// BEFORE FIX
const RALD_AUTH_URL = "https://accounts.rald.cloud"  // ← DEAD DOMAIN

// AFTER FIX (commit 5a2fa01)
const RALD_AUTH_URL = 
  (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) 
  ?? "https://profiles.rald.cloud"
```

**Impact:** Any auth flow initiated from within the messenger frontend would redirect to 403.

---

## ROOT CAUSE 2: TypeScript CI Failures

Three TS2322 errors prevented messenger CI from passing and thus blocked deployment verification:

| File | Error | Fix |
|------|-------|-----|
| auth.tsx:261 | RaldFrame prop `raldState` → should be `state` | commit 30cd7fc |
| auth.tsx:300,347 | HeartbeatButton missing required `onClick` prop | make `onClick` optional (commit bf10382) |
| heartbeat-button.tsx | No `type` or `raldState` props in interface | Added to interface (commit bf10382) |

---

## ROOT CAUSE 3: No Frontend Domain (CRITICAL — UNRESOLVED)

```
messenger.rald.cloud → Cloudflare Worker (API)
                     → There is NO frontend SPA domain
```

The Messenger frontend (Vite React SPA) is built via the "Deploy — Cloudflare Pages" CI job.  
This job exits with code 0 if `CLOUDFLARE_API_TOKEN` is missing in GitHub secrets.  
**Result:** CI appears to succeed, but the SPA is never actually deployed.

**Evidence:**
- `messenger.rald.cloud/health` → JSON response (API Worker)
- `messenger.rald.cloud/` → no SPA, no index.html, no React app

---

## TRACE: profiles.rald.cloud → auth.rald.cloud → messenger.rald.cloud

| Step | Status | Notes |
|------|--------|-------|
| User visits messenger domain | ❌ | No frontend to visit |
| SSO token request | ✅ | Would work if frontend existed |
| Messenger API JWT validation | ✅ | Confirmed working |
| Messenger API auth middleware | ✅ | Returns 401 on invalid JWT |
| Session persistence | ❓ | Cannot test without frontend |

---

## ACTIONS REQUIRED

1. **Add `CLOUDFLARE_API_TOKEN`** to messenger GitHub secrets
2. **Assign a frontend domain** for the Pages SPA (e.g. `chat.rald.cloud`)
3. **Configure Pages custom domain** in Cloudflare dashboard or via Wrangler
4. Push to messenger/main to trigger fresh deploy with all TS fixes applied

---

## VERDICT

| Component | Status |
|-----------|--------|
| Messenger API Worker | ✅ DEPLOYED & WORKING |
| Messenger Frontend SPA | ❌ NOT DEPLOYED |
| Auth URL Fix | ✅ Applied (commit 5a2fa01) |
| TS Errors Fixed | ✅ (commits bf10382, 30cd7fc) |
| Public User Access | ❌ BLOCKED until frontend domain assigned |
