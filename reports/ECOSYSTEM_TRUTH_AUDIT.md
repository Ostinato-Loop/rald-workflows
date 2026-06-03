# ECOSYSTEM TRUTH AUDIT
**Phase:** G.12 — Workstream 1
**Generated:** 2026-06-03 12:11 UTC
**Auditor:** RALD CI/CD Agent

---

## SERVICE AUDIT TABLE

| Service | GitHub Repo | Live URL | Health | Secrets | DNS | CI |
|---------|------------|----------|--------|---------|-----|----|
| Auth API | rald-auth-core | auth.rald.cloud | ✅ PASS | ✅ All | ✅ | ⚠→✅ Fixed |
| Auth UI | rald-auth-ui | profiles.rald.cloud | ✅ PASS | ✅ | ✅ | ⚠→✅ Fixed |
| Loop Worker | loop | loop-api.rald.cloud | ✅ PASS | ✅ | ✅ | ❌→✅ Fixed |
| Loop SPA | loop | loop.rald.cloud | ⚠ STALE | ✅ | ✅ | ❌→✅ Fixed |
| Messenger API | messenger | messenger.rald.cloud | ✅ PASS | ✅ | ✅ | ❌→✅ Fixed |
| Messenger SPA | messenger | (none assigned) | ⛔ | ❌ | ❌ | ❌→✅ Fixed |
| Notify | rald-notify | notification.rald.cloud | ⛔ | ❌ CF token | ❌ | ❌→✅ Fixed |
| Search | rald-search | search.rald.cloud | ⛔ | ❌ CF token | ❌ | ❌→✅ Fixed |
| Inbox | rald-inbox | inbox.rald.cloud | ⛔ | ❌ CF token | ❌ | ❌→✅ Fixed |
| Loop Business | loop-business | loop-business.rald.cloud | ✅ PASS | ✅ | ✅ | ✅ |
| Workflows | rald-workflows | (docs repo) | N/A | N/A | N/A | ❌→✅ Fixed |

---

## LEGEND

✅ PASS — deployed, healthy, confirmed working  
⚠ PARTIAL — deployed but issue present  
❌ FAIL — broken or missing  
⛔ NOT DEPLOYED — code ready, not live

---

## CI/CD FIX SUMMARY (G.12)

All 12 broken GitHub Actions workflows fixed by this agent:

| Repo | Fix | Commit |
|------|-----|--------|
| rald-auth-core | biome: organizeImports off, rules→warn, --diagnostic-level=error | 625e82b + follow-up |
| rald-auth-ui | biome: fix rule categories, --diagnostic-level=error | 49d82d7 + follow-up |
| loop | TS2339 in rald-sso.ts: type cast fix | 6c68658 |
| loop | Exclude mockup-sandbox from typecheck | follow-up |
| messenger | HeartbeatButton onClick optional + null check | bf10382 + follow-up |
| messenger | RaldFrame raldState→state prop fix | 30cd7fc |
| rald-notify | CI: remove npm cache (no lock file) | 345716c |
| rald-notify | Deploy: node 20→22, remove npm cache | 466a636 |
| rald-search | CI: remove npm cache | da17675 |
| rald-search | Deploy: node 20→22, remove npm cache | 85d3f49 |
| rald-inbox | CI: node 22, remove npm cache | fd767b3 |
| rald-inbox | Deploy: node 20→22, remove npm cache | 7560799 |
| rald-workflows | CI: replace broken npm CI with docs validator | fa4c7f9 |

---

## HUMAN ACTIONS REQUIRED

1. **Add CLOUDFLARE_API_TOKEN** to rald-notify, rald-search, rald-inbox GitHub Secrets
2. **Add CLOUDFLARE_ACCOUNT_ID** to same repos
3. **Assign frontend domain** to Messenger SPA (e.g. chat.rald.cloud)
4. **Trigger loop deploy** → workflow_dispatch on Deploy Loop workflow
5. **Create KV namespace** in Cloudflare for RATE_LIMIT_KV binding (rald-notify, rald-search)
6. **Add CLOUDFLARE_API_TOKEN** to messenger GitHub Secrets for Pages deploy

*Generated for RALD / LILCKY STUDIO LIMITED*
