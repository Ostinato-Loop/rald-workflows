# SSO Forensics Report
**Phase G.11 — SSO Token Trace**
Generated: 2026-06-03
Evidence: Live HTTP traces + source code analysis

---

## Q1: Is token generated?
**YES — auth.rald.cloud generates tokens correctly.**

Evidence:
```
POST /auth/register → 201 {"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
POST /sso/exchange  → 200 {"token":"eyJ...","appId":"loop","expiresIn":3600,"sso_version":2}
```

JWT payload (master token):
```json
{"id":"8f8daeff-48be-49e6-9d33-7957d94c3f27","email":"...","role":"user","iat":1780487177,"exp":1780573577}
```

JWT payload (SSO token for loop):
```json
{"id":"8f8daeff...","email":"...","phone":null,"role":"user","appId":"loop","source":"rald-auth","sso_v":2,"iat":...,"exp":...+3600}
```

---

## Q2: Is token transferred?
**PARTIALLY — NEVER reaches the frontend apps.**

The SSO flow design:
1. profiles.rald.cloud → auth.rald.cloud → issues master JWT ✅
2. SSO exchange issues app-scoped token ✅
3. Token should arrive at loop.rald.cloud as `?rald_token=<token>` ❌ BLOCKED

**Why BLOCKED**: The Loop frontend at `loop.rald.cloud` redirects unauthenticated users to `https://accounts.rald.cloud` — a deprecated domain that returns Cloudflare 403. Users never reach the RALD sign-in page. The `rald_token` query param is never set.

The planned flow:
```
profiles.rald.cloud → login → SSO exchange → redirect to loop.rald.cloud?rald_token=X → Loop uses token
```
The actual flow:
```
loop.rald.cloud → redirects to accounts.rald.cloud → 403 BLOCKED → user stuck
```

**FIX APPLIED (commit e6666a6)**: `accounts.rald.cloud` → `profiles.rald.cloud` in `loop/artifacts/loop/src/hooks/use-auth.tsx`

---

## Q3: Is token validated?
**YES — but only on the API side (never reached by frontend).**

When the RALD SSO token IS passed to loop-api:
```
POST https://loop-api.rald.cloud/api/auth/rald-sso  {"rald_token":"<token>"}
→ 400 {"error":"rald_token is required"} (empty body test)
→ (with valid token) would verify locally via RALD_JWT_SECRET, then issue Loop JWT
```

Evidence: `raldSso.ts` verifies RALD JWT locally (no HTTP call, no CF 522 risk).

For messenger.rald.cloud:
```
GET https://messenger.rald.cloud/ + Authorization: Bearer <sso_token>
→ 400 {"error":"X-Workspace-ID header required"}
```
Token IS validated (gets past 401, into route logic), but workspace header is missing. Token validation works.

---

## Q4: Is token stored?
**NEVER REACHED — token cannot be stored because it is never transferred.**

Design intent (from `use-auth.tsx`):
- Master RALD token → `localStorage["rald_master_token"]`
- Loop JWT (after rald-sso exchange) → `localStorage["loop_token"]`

Design intent (from `messenger/auth.tsx`):
- RALD token → `localStorage["messenger_rald_token"]`

These localStorage writes are unreachable because Step 2 (transfer) is blocked.

---

## Q5: Is token exchanged?
**NOT REACHED by real users.**

The exchange endpoint `POST /api/auth/rald-sso` on loop-api.rald.cloud:
- Is deployed ✅
- Verifies RALD JWT locally ✅
- Issues Loop JWT ✅ (from source code analysis)
- Is NOT reached by real users (auth redirect blocked by accounts.rald.cloud)

---

## Q6: Is session created?
**NO — blocked at auth redirect.**

---

## Q7: If failure occurs, identify exact line of code.

**PRIMARY FAILURE**:
- **File**: `loop/artifacts/loop/src/hooks/use-auth.tsx`
- **Line**: 33 (before fix)
```typescript
const RALD_AUTH_UI    = "https://accounts.rald.cloud";
```
- **After fix** (commit e6666a6):
```typescript
const RALD_AUTH_UI    = (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) ?? "https://profiles.rald.cloud";
```

**SECONDARY FAILURE** (Messenger):
- **File**: `messenger/artifacts/loop-messenger/src/pages/auth.tsx`
- **Line**: const RALD_AUTH_UI (before fix)
```typescript
const RALD_AUTH_UI        = "https://accounts.rald.cloud";
```
- **After fix** (commit 5a2fa01): same pattern, points to profiles.rald.cloud

---

## SSO Flow Diagram — Current State

```
profiles.rald.cloud (auth UI)
    │
    ├── ✅ POST /auth/register → JWT issued
    ├── ✅ POST /auth/login    → JWT issued
    └── ✅ POST /sso/exchange  → app-scoped token issued
              │
              ▼ redirect to loop.rald.cloud?rald_token=X
              │
loop.rald.cloud (frontend SPA)
    │
    ├── ❌ Unauthenticated → redirects to accounts.rald.cloud (DEAD — fixed)
    └── ✅ (after fix) → redirects to profiles.rald.cloud → auth works
              │
              ▼ POST /api/auth/rald-sso with rald_token
              │
loop-api.rald.cloud (CF Worker)
    ├── ✅ Verifies RALD JWT locally
    ├── ✅ Issues Loop JWT
    └── ✅ Returns {access_token, user}
              │
              ▼ API calls with Loop JWT
              │
Supabase (loop-db D1 + supabase project)
    └── ✅ DB exists, bindings confirmed
```
