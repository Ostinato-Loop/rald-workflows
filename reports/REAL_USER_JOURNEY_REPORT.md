# REAL USER JOURNEY REPORT
**Phase:** G.12 — Workstream 8
**Generated:** 2026-06-03 12:11 UTC
**Method:** Live API testing against production endpoints

---

## TEST RESULTS SUMMARY

| Step | Action | Result | Status |
|------|--------|--------|--------|
| 1 | Register (OTP request) | 200 OK, OTP delivered | ✅ PASS |
| 2 | OTP Login | 200 OK, RALD JWT issued | ✅ PASS |
| 3 | Logout | 200 OK, session revoked | ✅ PASS |
| 4 | Login again | 200 OK, new JWT issued | ✅ PASS |
| 5 | Open Loop | ⚠ Fix deployed, awaiting Pages rebuild | ⚠ PARTIAL |
| 6 | Open Messenger | ❌ No frontend deployed | ❌ FAIL |
| 7 | Navigate between apps | ❌ SSO redirect issues | ❌ FAIL |
| 8 | Revoke session | ✅ API confirms revocation | ✅ PASS |
| 9 | Login from second device | ✅ Independent session created | ✅ PASS |
| 10 | Recover account | ✅ New OTP flow works | ✅ PASS |

**PASS: 6/10 | PARTIAL: 1/10 | FAIL: 3/10**

---

## DETAILED TEST TRACE

### Step 1 — Register (POST /auth/send-otp)
```bash
curl -X POST https://auth.rald.cloud/auth/send-otp \
  -H "Content-Type: application/json" \
  -d '{"phone": "+2348XXXXXXXXX"}'

→ 200 OK
→ {"success": true, "message": "OTP sent"}
→ SMS delivered via Termii
```
**Result:** ✅ PASS

### Step 2 — OTP Login (POST /auth/verify-otp)
```bash
curl -X POST https://auth.rald.cloud/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"phone": "+2348XXXXXXXXX", "otp": "XXXXXX"}'

→ 200 OK
→ {"token": "<RALD_JWT>", "user": {"id": "...", "phone": "..."}}
```
**Result:** ✅ PASS

### Step 3 — Logout (POST /auth/logout)
```bash
curl -X POST https://auth.rald.cloud/auth/logout \
  -H "Authorization: Bearer <RALD_JWT>"

→ 200 OK
→ Session revoked in Supabase
```
**Result:** ✅ PASS

### Step 4 — Login Again
Repeat Step 2. New JWT issued, independent of previous session.
**Result:** ✅ PASS

### Step 5 — Open Loop (loop.rald.cloud)
- Auth redirect URL bug fixed (commit e6666a6)
- Fix awaiting Cloudflare Pages rebuild before going live
- SSO exchange endpoint functional: POST loop-api.rald.cloud/api/auth/rald-sso ✅
**Result:** ⚠ PARTIAL — Code fixed, deployment pending

### Step 6 — Open Messenger
- messenger.rald.cloud serves API only (no frontend SPA)
- No user-accessible Messenger interface exists
**Result:** ❌ FAIL — Frontend not deployed

### Step 7 — Navigate Between Apps
- Blocked by Messenger frontend not existing
- Loop→Messenger navigation has no target
**Result:** ❌ FAIL

### Step 8 — Revoke Session
Session revocation via POST /auth/logout confirmed working.
**Result:** ✅ PASS

### Step 9 — Login From Second Device
Tested with second OTP flow: new RALD_JWT issued, independent session.
**Result:** ✅ PASS

### Step 10 — Recover Account
Phone-based recovery: re-trigger OTP to same number. Works.
**Result:** ✅ PASS

---

## BLOCKING ISSUES FOR FULL PASS

1. **Loop Pages rebuild** — trigger deploy to apply auth URL fix
2. **Messenger frontend domain** — assign chat.rald.cloud to Pages SPA
3. **Messenger CLOUDFLARE_API_TOKEN** — add to GitHub Secrets

---

## OVERALL VERDICT

**PASS:** 6/10 steps  
**Classification:** NOT READY FOR PUBLIC LAUNCH  
**Minimum for Internal Testing:** Deploy messenger frontend, confirm loop Pages rebuild

*Generated for RALD / LILCKY STUDIO LIMITED*
