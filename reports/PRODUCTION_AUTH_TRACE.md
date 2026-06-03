# Production Auth Trace
**Phase G.11 — Live Evidence**
Generated: 2026-06-03T11:46–11:48 UTC
Method: curl against live production endpoints

---

## STEP 1 — Register test user
**URL**: POST https://auth.rald.cloud/auth/register
**Body**: `{"email":"g11test_1780487176@rald-forensics.dev","password":"...","name":"G11 Forensics"}`
**Status**: 201 CREATED
**Result**: ✅ JWT token issued. User ID `8f8daeff-48be-49e6-9d33-7957d94c3f27` created in Supabase.

## STEP 2 — Verify JWT
**URL**: GET https://auth.rald.cloud/me
**Auth**: Bearer <token>
**Status**: 200
**Result**: ✅ `{"id":"8f8daeff-48be-49e6-9d33-7957d94c3f27","rald_id":"RALD-8F8DAEFF","email":"g11test_1780487176@rald-forensics.dev","name":"G11 Forensics","role":"user","identity_hub":"profiles.rald.cloud"}`

## STEP 3 — SSO exchange for Loop
**URL**: POST https://auth.rald.cloud/sso/exchange
**Body**: `{"appId":"loop","redirect_to":"https://loop.rald.cloud"}`
**Status**: 200
**Result**: ✅ App-scoped token issued. `{"token":"eyJ...","appId":"loop","expiresIn":3600,"sso_version":2}`

## STEP 4 — SSO exchange for Messenger
**URL**: POST https://auth.rald.cloud/sso/exchange
**Body**: `{"appId":"messenger","redirect_to":"https://messenger.rald.cloud"}`
**Status**: 200
**Result**: ✅ App-scoped token issued. `{"token":"eyJ...","appId":"messenger","expiresIn":3600,"sso_version":2}`

## STEP 5 — Access loop.rald.cloud with token
**URL**: GET https://loop.rald.cloud/?rald_token=<loop_token>
**Status**: 200
**Result**: ⚠️ SPA HTML served. The page loads — BUT internally the SPA checks `localStorage` for `loop_token` and, finding none, redirects the browser to `https://accounts.rald.cloud?redirect_to=...` which is **inaccessible** (returns Cloudflare 403 + JS challenge). The page appears to load but the user immediately bounces to a dead domain.

## STEP 6 — What does accounts.rald.cloud return?
**URL**: GET https://accounts.rald.cloud
**Status**: 403
**Result**: ❌ Cloudflare "managed challenge" page (JavaScript required, bot challenge). For real users: they get a blank challenge page and cannot proceed. For automated agents: 403. **This domain is effectively dead for auth purposes.**

## STEP 7 — Access messenger.rald.cloud root
**URL**: GET https://messenger.rald.cloud/
**Status**: 401
**Result**: ❌ `{"error":"Unauthorized"}` — This is the backend API Worker, not a frontend. No UI is served.

## STEP 8 — Access messenger.rald.cloud with SSO token as Bearer
**URL**: GET https://messenger.rald.cloud/
**Auth**: Bearer <messenger_sso_token>
**Status**: 400
**Result**: ❌ `{"error":"X-Workspace-ID header required"}` — Token is accepted but workspace header is missing. Even with a valid token, the root has no frontend to serve.

## STEP 9 — messenger.rald.cloud/health
**URL**: GET https://messenger.rald.cloud/health
**Status**: 200
**Result**: ✅ `{"status":"ok","service":"loop-messenger-api","version":"1.0.0","environment":"production"}`

## STEP 10 — loop-api.rald.cloud SSO bridge
**URL**: POST https://loop-api.rald.cloud/api/auth/rald-sso
**Body**: `{"rald_token":"<sso_token>"}`
**Status**: 200
**Result**: ✅ SSO bridge responds. Worker is live. But no `/api/auth/me`, `/api/users`, or `/api/conversations` routes are registered.

---

## Failure Point
**FAILURE AT STEP 5** — The user receives a valid SSO token but the Loop SPA unconditionally redirects to `https://accounts.rald.cloud` (deprecated, unreachable) before it can use that token.

**Root file**: `loop/artifacts/loop/src/hooks/use-auth.tsx`, line 33
```typescript
const RALD_AUTH_UI = "https://accounts.rald.cloud"; // BROKEN — hardcoded, deprecated
```
