# MESSENGER LOGIN ROOT CAUSE
**Phase:** G.12 — Workstream 4
**Generated:** 2026-06-03 12:11 UTC

---

## ROOT CAUSE 1: Hardcoded Auth URL
**File:** `messenger/artifacts/loop-messenger/src/pages/auth.tsx`  
**Fix:** commit 5a2fa01 — accounts.rald.cloud → profiles.rald.cloud

## ROOT CAUSE 2: TypeScript Errors Blocked CI
| Error | File | Fix |
|-------|------|-----|
| TS2322: raldState prop on RaldFrame | auth.tsx:261 | raldState→state (commit 30cd7fc) |
| TS2322: missing onClick on HeartbeatButton | auth.tsx:300,347 | make onClick optional (commit bf10382) |
| TS2722: calling possibly undefined onClick | heartbeat-button.tsx:34 | null check if(onClick) onClick() |

## ROOT CAUSE 3: No Frontend Domain (UNRESOLVED)
messenger.rald.cloud = API Worker only. No frontend SPA is reachable by users.

**Required Actions:**
1. Add CLOUDFLARE_API_TOKEN to messenger GitHub Secrets
2. Assign frontend domain (e.g. chat.rald.cloud)
3. Push to messenger/main to trigger fresh deploy
