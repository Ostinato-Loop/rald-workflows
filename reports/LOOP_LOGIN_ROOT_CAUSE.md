# LOOP LOGIN ROOT CAUSE
**Phase:** G.12 — Workstream 3
**Generated:** 2026-06-03 12:11 UTC

---

## ROOT CAUSE: Hardcoded Dead Domain

**File:** `loop/artifacts/loop/src/hooks/use-auth.tsx` line 33  
**Impact:** 100% of unauthenticated Loop users blocked  
**Status:** FIXED (commit e6666a6)

```typescript
// BEFORE (broken)
const RALD_AUTH_UI = "https://accounts.rald.cloud"

// AFTER (fixed)
const RALD_AUTH_UI = (import.meta.env.VITE_RALD_AUTH_URL as string | undefined) 
  ?? "https://profiles.rald.cloud"
```

`accounts.rald.cloud` returns HTTP 403 (Cloudflare managed challenge) for all users.  
`profiles.rald.cloud` returns HTTP 200 — correct auth entry point.

## SECONDARY: TypeScript CI Failure

**File:** `loop/artifacts/cloudflare-worker/src/routes/rald-sso.ts` lines 78+81  
**Error:** TS2339 Property rald_token does not exist on union type  
**Fix:** Type cast `(await c.req.json().catch(() => ({}))) as { rald_token?: string }`  
**Commit:** 6c68658

## REMAINING: Pages Rebuild Required

Loop.rald.cloud still serves the pre-fix bundle.  
Action: Trigger workflow_dispatch on "Deploy Loop" workflow.
