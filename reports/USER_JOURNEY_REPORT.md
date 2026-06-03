# User Journey Report
**Phase G.11 — Real User Experience Analysis**
Generated: 2026-06-03
Method: Source code analysis + live HTTP trace

---

## Journey 1: New user trying to enter Loop (loop.rald.cloud)

### Step-by-step trace

| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User navigates to `https://loop.rald.cloud` | SPA HTML loads (200), React hydrates | ✅ |
| 2 | React renders, checks `localStorage["loop_token"]` | Empty (first visit) | — |
| 3 | Auth hook calls `redirectToRaldAuth(appUrl, "loop", "/")` | Constructs redirect URL using `RALD_AUTH_UI` const | — |
| 4 | Browser navigates to `https://accounts.rald.cloud?redirect_to=...&app_id=loop` | **Cloudflare Managed Challenge (403)** — user sees blank page with JS challenge | ❌ FAIL |
| 5 | User cannot pass the challenge (not a bot) | Browser may retry or show error | ❌ DEAD END |

### Root cause
`RALD_AUTH_UI` hardcoded to `"https://accounts.rald.cloud"` (deprecated domain).

### Fix applied
Commit `e6666a6` — `loop/artifacts/loop/src/hooks/use-auth.tsx`:
```typescript
// BEFORE (broken):
const RALD_AUTH_UI = "https://accounts.rald.cloud";

// AFTER (fixed):
const RALD_AUTH_UI = (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) ?? "https://profiles.rald.cloud";
```

### Expected journey AFTER fix (next deploy)
| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User navigates to `https://loop.rald.cloud` | SPA loads | ✅ |
| 2 | Auth hook: no token in localStorage | Redirect triggered | — |
| 3 | Browser → `https://profiles.rald.cloud?redirect_to=https://loop.rald.cloud/&app_id=loop` | RALD Auth UI loads | ✅ |
| 4 | User signs in (OTP/email) at profiles.rald.cloud | Auth issues RALD JWT + SSO token | ✅ |
| 5 | Redirect back: `https://loop.rald.cloud/?rald_token=<TOKEN>&app_id=loop` | Loop SPA receives token | ✅ |
| 6 | Loop SPA: POST `/api/auth/rald-sso` to loop-api.rald.cloud | Loop JWT issued | ✅ |
| 7 | Loop JWT stored in localStorage["loop_token"] | User session established | ✅ |
| 8 | Loop app loads feed, rooms, trending | Requires `/api/trending`, `/api/rooms` | ✅ |

---

## Journey 2: New user trying to enter Messenger (messenger.rald.cloud)

### Step-by-step trace

| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User navigates to `https://messenger.rald.cloud` | **401 `{"error":"Unauthorized"}`** — JSON response, NO frontend | ❌ FAIL |
| 2 | No SPA exists at this URL | Nothing to render | ❌ DEAD END |

### Root cause
`messenger.rald.cloud` is bound to the **backend API Worker** (`workers/loop-messenger-api`), not the frontend SPA. The messenger frontend (`artifacts/loop-messenger`) exists in the repo but was **never deployed**. The `deploy-pages.yml` CI workflow silently skips deployment when `CLOUDFLARE_API_TOKEN` is not configured in GitHub repo secrets.

### Evidence
```
GET https://messenger.rald.cloud/
→ HTTP 401
→ Content-Type: application/json
→ Body: {"error":"Unauthorized"}
```

The Worker has `authMiddleware` applied to ALL routes except `/health`. The root path `/` is just another protected API route — there is no route that serves static HTML.

### What would fix Messenger?

**Option A — Dedicated subdomain for frontend** (recommended):
1. Deploy messenger frontend to Cloudflare Pages (project: `loop-messenger`)
2. Add GitHub secret `CLOUDFLARE_API_TOKEN` with Pages:Edit permission to the `messenger` repo
3. Configure a custom domain for the Pages deployment (e.g., `app.messenger.rald.cloud` or `chat.rald.cloud`) — `messenger.rald.cloud` is already taken by the Worker
4. Update messenger frontend's `VITE_API_URL` to `https://messenger.rald.cloud` (API)

**Option B — Route static files from Worker**:
Serve the Vite bundle as static assets from the Worker itself using Workers Sites or Assets. Not recommended (complexity).

---

## Journey 3: Existing Loop user navigating to Messenger (cross-app SSO)

### Before fix
| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User clicks "Open Messenger" in Loop | Calls `openMessenger()` | — |
| 2 | No `rald_master_token` in localStorage (never set, auth was blocked) | Falls back to `redirectToRaldAuth()` | — |
| 3 | Redirect to `accounts.rald.cloud` | ❌ 403 BLOCKED | ❌ FAIL |

### After Loop fix (pending next deploy)
Assuming user has a valid Loop session:
| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User clicks "Open Messenger" in Loop | Calls `openMessenger()` | — |
| 2 | `rald_master_token` present in localStorage | Constructs SSO URL | ✅ |
| 3 | Browser → `https://messenger.rald.cloud/chats?rald_token=<TOKEN>&app_id=messenger` | 401 — no frontend exists | ❌ FAIL |

**Cross-app SSO logic works, but there is no Messenger frontend to receive the token.**

---

## Journey 4: profiles.rald.cloud standalone (working)

| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | User navigates to `https://profiles.rald.cloud` | SPA loads (200) | ✅ |
| 2 | Auth UI shown (sign in with OTP / email) | RALD Auth form renders | ✅ |
| 3 | User submits → POST `https://auth.rald.cloud/auth/register` | 201 JWT | ✅ |
| 4 | Profile created in Supabase | User can browse profiles | ✅ |

**profiles.rald.cloud is the only fully functional user-facing app.**
