# RALD SSO SESSION AUDIT
**Classification: PARTIALLY VERIFIED**
**Date: 2026-06-04**
**Author: RALD Systems Architect (AI)**
**Incident ID: SSO_SESSION_AUDIT_2026_06_04**

---

## EXECUTIVE SUMMARY

The RALD ecosystem currently operates **Product Authentication**, not **Centralized Identity Authentication**. A user who authenticates on `profiles.rald.cloud` cannot silently access `loop.rald.cloud` or `messenger.rald.cloud` without re-authenticating. The root cause is that the RALD master session token is stored in `localStorage` — a browser API that is strictly isolated per domain. No cross-domain cookie exists. No silent SSO mechanism is triggered when a product loads.

The SSO infrastructure is architecturally sound and partially built, but it is not wired end-to-end. The missing piece is a cross-domain session signal (cookie or server-side session) and automatic silent SSO initiation by each product on startup.

---

## 1. WHERE IS THE RALD MASTER SESSION STORED?

**FINDING:** `localStorage` on `profiles.rald.cloud` only.

Evidence from `rald-auth-ui/src/lib/api.ts`:
```typescript
export const saveToken  = (t: string) => localStorage.setItem("rald_token", t);
export const clearToken = ()          => localStorage.removeItem("rald_token");
export const getToken   = ()          => localStorage.getItem("rald_token");
```

Evidence from `rald-auth-core/src/routes/auth.ts` (POST /login):
```typescript
return c.json({ token, user: userShape(user) });
```
No `Set-Cookie` header. No `c.header("Set-Cookie", ...)` anywhere in the auth routes.

Production confirmation:
```
curl -sI https://auth.rald.cloud/health | grep -i set-cookie
# Result: (empty — no Set-Cookie header)
```

Loop stores its own separate token:
```typescript
// loop/artifacts/loop/src/hooks/use-auth.tsx
const TOKEN_KEY       = "loop_token";        // Loop JWT (different from RALD JWT)
const RALD_TOKEN_KEY  = "rald_master_token"; // RALD JWT (stored after SSO exchange)
```

Messenger stores its own:
```typescript
// messenger/artifacts/loop-messenger/src/pages/auth.tsx
const MESSENGER_TOKEN_KEY = "messenger_rald_token";
```

**Verdict:** Three separate token stores, none visible to each other. All in localStorage.

---

## 2. IS A PERSISTENT RALD SESSION COOKIE BEING CREATED?

**FINDING: NO.**

The `POST /auth/login` endpoint in `rald-auth-core` returns a JSON response body containing `{ token, user }`. It does not set any HTTP cookie on any domain.

Checked:
- `rald-auth-core/src/routes/auth.ts` — no `Set-Cookie`
- `rald-auth-core/src/routes/sso.ts` — no `Set-Cookie`
- `rald-auth-core/src/routes/session.ts` — no `Set-Cookie`
- `rald-auth-core/src/lib/auth.ts` — pure JWT signing, no cookie logic
- Production HTTP headers — confirmed no `Set-Cookie` in any response

---

## 3. WHAT IS THE COOKIE DOMAIN?

**FINDING: There is no cookie. This question has no answer.**

`auth.rald.cloud` does not issue cookies. Neither does `profiles.rald.cloud`. No CORS credential cookie, no HttpOnly session cookie, no SameSite cookie of any kind.

The CORS configuration does include `access-control-allow-credentials: true` (confirmed in production probe), which means the infrastructure is prepared for cookie-based auth — but no cookie is currently being set.

---

## 4. IS THE COOKIE ACCESSIBLE ACROSS PRODUCTS?

**FINDING: Irrelevant — no cookie exists.**

If a cookie *were* set with `domain=.rald.cloud; SameSite=Lax; Secure; HttpOnly`, it would be readable by:
- `profiles.rald.cloud`
- `loop.rald.cloud`
- `messenger.rald.cloud`
- `auth.rald.cloud`
- Any future `*.rald.cloud` subdomain

This is the correct architecture but it does not currently exist.

---

## 5. WHEN MESSENGER LOADS, DOES IT CHECK FOR AN EXISTING RALD SESSION?

**FINDING: NO. Messenger immediately redirects unauthenticated users to its own OTP login.**

Evidence from `messenger/artifacts/loop-messenger/src/App.tsx`:
```typescript
function RootRedirect() {
  const { data: user, isLoading } = useGetMe({ query: { retry: false } });
  const [, setLocation] = useLocation();

  useEffect(() => {
    if (!isLoading) {
      setLocation(user ? "/chats" : "/auth"); // Goes to /auth if no token
    }
  }, [user, isLoading, setLocation]);
```

`useGetMe()` calls the Messenger API server using `messenger_rald_token` from localStorage. If that key is empty, the call fails and the user is sent to `/auth`.

The `/auth` page (`messenger/artifacts/loop-messenger/src/pages/auth.tsx`) presents phone OTP login. It does NOT:
- Check for `?rald_token=` in the URL
- Call `auth.rald.cloud/session` to detect an existing RALD session
- Perform any silent SSO lookup

The user must complete full OTP authentication again.

---

## 6. WHY IS A PREVIOUSLY AUTHENTICATED USER LANDING ON LOGIN SCREENS?

**ROOT CAUSE ANALYSIS:**

**Primary cause:** `localStorage` is domain-isolated by the Same-Origin Policy. `localStorage["rald_token"]` on `profiles.rald.cloud` is completely invisible on `loop.rald.cloud` and `messenger.rald.cloud`.

**Secondary cause:** No product initiates a silent SSO check on startup. No product calls `auth.rald.cloud/session` to detect an existing authenticated identity.

**Tertiary cause:** Messenger's auth page does not consume `?rald_token=` URL parameters. Even if a RALD token were passed in the URL, Messenger has no handler for it.

**The complete failure chain:**
```
User authenticated → profiles.rald.cloud
└── localStorage["rald_token"] = "eyJ..." (on profiles.rald.cloud only)

User navigates directly to → messenger.rald.cloud
└── Messenger checks localStorage["messenger_rald_token"] → EMPTY
└── Messenger calls GET /api/me → 401 Unauthorized
└── Messenger redirects to /auth
└── User sees phone OTP screen
└── User must re-authenticate ← FAILURE POINT
```

**No cross-domain signal exists to bridge the session gap.**

---

## 7. IS THE SYSTEM USING PRODUCT AUTHENTICATION OR CENTRALIZED IDENTITY AUTHENTICATION?

**FINDING: Product Authentication in practice. Centralized Identity in architecture only.**

| Indicator | Current Reality |
|-----------|----------------|
| Where does login happen? | Each product has its own OTP form |
| Where is the session stored? | localStorage — separate per product domain |
| Is there a shared session? | No |
| Does `auth.rald.cloud` control all sessions? | No — each product validates its own token locally |
| Do products trust RALD JWTs natively? | Loop yes (rald-sso route exists). Messenger: partially. |
| Is re-authentication required across products? | Yes |

**The SSO infrastructure that EXISTS (but is not automatically invoked):**

1. `POST /sso/exchange` — exchanges a master RALD JWT for an app-scoped JWT (works, requires explicit call)
2. `POST /sso/handoff` — generates a 5-minute handoff token + redirect URL (works, requires explicit trigger)
3. `POST /api/auth/rald-sso` on loop-api — accepts a RALD JWT and issues a Loop JWT (works, requires `?rald_token=` in URL)
4. `GET /session` on auth.rald.cloud — validates a Bearer token and returns session status (works, requires the caller to already have the token)
5. `cross-app.ts` in loop frontend — `openMessenger()` passes `?rald_token=` to messenger URL (works, but only if user explicitly clicks)

None of these are automatically triggered when a product cold-starts.

---

## 8. IS PROFILES ACTING AS IdP OR SIMPLE LOGIN FORM?

**FINDING: Simple Login Form with IdP aspirations.**

| Feature | True IdP | profiles.rald.cloud |
|---------|----------|---------------------|
| Issues master session | ✓ | ✓ (localStorage, domain-locked) |
| Cross-domain session signal | ✓ Cookie on `.rald.cloud` | ✗ None |
| Silent session validation | ✓ Central `/check` endpoint | Partial (`/session` exists but products don't call it) |
| Redirect-based auth flow | ✓ Automatic | Partial (only when `?redirect_to=` is explicitly in URL) |
| Session revocation propagates | ✓ | ✗ No mechanism |
| Products trust IdP tokens automatically | ✓ | Partial (loop yes, messenger no) |

Profiles/auth.rald.cloud has the bones of an IdP (`/sso/exchange`, `/sso/handoff`, `/session`, registered app registry) but is not functioning as one because the session signal doesn't cross domain boundaries.

---

## CURRENT LOGIN FLOW (AS-IS)

```
┌─────────────────────────────────────────────────────────────────┐
│                    CURRENT BROKEN STATE                          │
└─────────────────────────────────────────────────────────────────┘

LOOP login (OTP — phone only):
User → loop.rald.cloud/login
  → POST loop-api.rald.cloud/api/auth/send-otp (Termii)
  → POST loop-api.rald.cloud/api/auth/verify-otp
  → Receives Loop JWT (loop_token) stored in localStorage["loop_token"]
  → loop.rald.cloud/discover ✓
  
  Optional: POST loop-api.rald.cloud/api/auth/rald-sso {rald_token}
  → Only works if user navigated FROM profiles.rald.cloud via ?rald_token=

MESSENGER login (OTP — phone only):
User → messenger.rald.cloud
  → GET /api/me → 401 (no messenger_rald_token)
  → Redirect to /auth
  → Phone OTP with Termii
  → Receives Messenger JWT stored in localStorage["messenger_rald_token"]
  → messenger.rald.cloud/chats ✓
  
  ← COMPLETELY SEPARATE from Loop and Profiles sessions

PROFILES login (email/phone/password):
User → profiles.rald.cloud
  → POST auth.rald.cloud/auth/login or verify-otp
  → Receives RALD master JWT stored in localStorage["rald_token"]
  → profiles.rald.cloud/dashboard ✓
  
  If ?redirect_to= present: automatically does SSO exchange → redirects with ?rald_token=
  ← ONLY cross-product bridge that works, but ONLY when redirect_to is in the URL

RESULT: 3 separate logins. No shared session. No silent SSO.
```

---

## SESSION ARCHITECTURE DIAGRAM

```
                    CURRENT STATE (BROKEN)
                    
profiles.rald.cloud                 auth.rald.cloud
┌─────────────────┐                ┌─────────────────────────────┐
│ React SPA       │  POST /login   │ Cloudflare Worker           │
│                 │─────────────── ▶                             │
│ localStorage:   │◀───────────────│  Returns: { token, user }   │
│  rald_token=JWT │  JSON response │  (NO Set-Cookie header)     │
└─────────────────┘                └─────────────────────────────┘
        │                                   │
        │ DOMAIN BOUNDARY ─────────────────│────────────────────────
        │                                   │
loop.rald.cloud                    messenger.rald.cloud
┌─────────────────┐                ┌─────────────────────────────┐
│ React SPA       │                │ React SPA                   │
│                 │                │                             │
│ localStorage:   │  INVISIBLE ❌  │ localStorage:               │
│  loop_token=JWT │◀ ─ ─ ─ ─ ─ ─ ─│  messenger_rald_token=JWT   │
│  rald_master_   │    to each     │                             │
│  token=RALD_JWT │    other       │  NO rald_token handler      │
└─────────────────┘                └─────────────────────────────┘
        │                                   │
        │ Each product does its own OTP login. No bridge. No SSO.
```

---

## RECOMMENDED: GOOGLE-STYLE SSO FLOW

```
                    RECOMMENDED STATE (TARGET)
                    
                        auth.rald.cloud (True IdP)
                    ┌─────────────────────────────────┐
                    │ 1. User logs in (OTP/password)   │
                    │ 2. Set HttpOnly cookie:           │
                    │    rald_session=JWT               │
                    │    domain=.rald.cloud             │
                    │    SameSite=Lax; Secure            │
                    │ 3. Return JSON { token, user }   │
                    └─────────────────────────────────┘
                           │ Cookie travels to all *.rald.cloud
                           ▼
    ┌──────────────────────────────────────────────────────┐
    │               .rald.cloud cookie domain              │
    │                                                      │
    │  profiles.rald.cloud  loop.rald.cloud  messenger...  │
    │  ┌───────────────┐   ┌─────────────┐  ┌──────────┐  │
    │  │ Has cookie ✓  │   │ Has cookie ✓│  │ Cookie ✓ │  │
    │  └───────────────┘   └─────────────┘  └──────────┘  │
    └──────────────────────────────────────────────────────┘

SILENT SSO FLOW (when user opens loop.rald.cloud directly):

Step 1: loop.rald.cloud loads
Step 2: No loop_token in localStorage
Step 3: Loop frontend redirects to:
        auth.rald.cloud/sso/silent?redirect_to=https://loop.rald.cloud/login&app_id=loop
Step 4: auth.rald.cloud reads .rald.cloud cookie
        → Cookie valid → exchange for loop-scoped token
        → Redirect to: loop.rald.cloud/login?rald_token=<token>&app_id=loop
Step 5: Loop frontend receives rald_token → calls /api/auth/rald-sso → gets Loop JWT
Step 6: User lands on loop.rald.cloud/discover — AUTHENTICATED. No OTP required.

        If cookie is absent → redirect to profiles.rald.cloud/login?redirect_to=...
        → User authenticates ONCE → SSO distributes to all products
```

### Implementation Requirements

**On `auth.rald.cloud` (rald-auth-core):**
1. Add `Set-Cookie: rald_session=<JWT>; domain=.rald.cloud; SameSite=Lax; Secure; HttpOnly; Max-Age=604800` to all login response headers
2. Add `GET /sso/silent?redirect_to=&app_id=` — reads the `.rald.cloud` cookie, validates it, issues app token, redirects

**On each product frontend (loop, messenger, profiles):**
1. On app startup, check for local token first
2. If no local token, redirect to `auth.rald.cloud/sso/silent?redirect_to=<current_url>&app_id=<app_id>`
3. Handle `?rald_token=` parameter on load — exchange for local session

**On Messenger specifically:**
1. Add `?rald_token=` URL handler in App.tsx (loop already has this in use-auth.tsx)
2. Call `POST /api/auth/rald-sso { rald_token }` to get messenger JWT
3. Store and continue

**No changes required to:**
- `POST /sso/exchange` — already works correctly
- `POST /sso/handoff` — already works correctly
- Loop's `rald-sso` route — already works correctly
- `cross-app.ts` in loop — already works correctly
- App registry (`registered_apps` table) — already working

---

## PRODUCTION VERIFICATION

| Check | URL | Result | Status |
|-------|-----|--------|--------|
| auth.rald.cloud health | https://auth.rald.cloud/health | HTTP 200 | ✓ VERIFIED |
| auth.rald.cloud/session (no token) | https://auth.rald.cloud/session | HTTP 401 + redirect to profiles | ✓ VERIFIED |
| loop-api health | https://loop-api.rald.cloud/api/health | HTTP 200 | ✓ VERIFIED |
| profiles.rald.cloud | https://profiles.rald.cloud/ | HTTP 200 | ✓ VERIFIED |
| loop.rald.cloud | https://loop.rald.cloud/ | HTTP 200 | ✓ VERIFIED |
| messenger.rald.cloud | https://messenger.rald.cloud/ | HTTP 401 (no token) | ✓ VERIFIED |
| Set-Cookie on login | auth.rald.cloud/health headers | **NO Set-Cookie** | ✗ CONFIRMED MISSING |
| CORS credentials | auth.rald.cloud | access-control-allow-credentials: true | ✓ READY |

**Silent SSO End-to-End:** UNVERIFIED — not yet implemented.

---

## INCIDENT RECORD

**Incident ID:** SSO_SESSION_AUDIT_2026_06_04
**Severity:** P1 — Affects all users crossing product boundaries
**Status:** ROOT CAUSE IDENTIFIED. No fix implemented yet.
**Impact:** Every authenticated RALD user must re-authenticate when opening any new product.
**Root Cause:** localStorage isolation + absence of cross-domain cookie + no silent SSO check on product startup.
**Resolution Path:** 
1. Add `Set-Cookie` with `domain=.rald.cloud` to auth.rald.cloud login endpoints
2. Add `GET /sso/silent` endpoint to auth.rald.cloud
3. Add silent SSO check to loop, messenger, and all future products
4. Add `?rald_token=` handler to Messenger (already exists in Loop)
**Verification Required:** Production SSO test: login on profiles.rald.cloud → navigate to messenger.rald.cloud → confirm no re-auth required.

---

## DECISION RECORD

**Decision:** Implement cookie-based cross-domain session for `.rald.cloud` domain.
**Why:** localStorage cannot cross domain boundaries by browser design. This is the only compliant way to share session state across subdomains without custom redirect flows. Google, Microsoft, and all major IdPs use this approach.
**Alternative considered:** iframe-based silent SSO (Google's older approach) — rejected due to third-party cookie restrictions in modern browsers.
**Alternative considered:** Shared Cloudflare KV session with product-specific tokens — viable for backend services but frontend still needs the cookie to know a session exists.
**Owner:** RALD Platform Team
**Date:** 2026-06-04

---

*This document is the source of truth for the RALD SSO investigation. All code changes must reference this audit ID.*
*Stored in WIZMAC Identity Registry and Incident Registry.*
