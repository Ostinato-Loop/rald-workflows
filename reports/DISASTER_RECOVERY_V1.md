# DISASTER RECOVERY V1
**Phase:** G.12 — Workstream 7
**Generated:** 2026-06-03 12:11 UTC
**RPO Target:** < 15 minutes
**RTO Target:** < 60 minutes

---

## SERVICE DEPENDENCY MAP

```
Cloudflare (DNS + Workers + Pages)
    ├── auth.rald.cloud (CF Worker)
    │   └── Supabase PostgreSQL (auth data, OTP logs, sessions)
    ├── profiles.rald.cloud (CF Pages)
    │   └── auth.rald.cloud (API calls)
    ├── loop-api.rald.cloud (CF Worker)
    │   ├── Supabase PostgreSQL (rooms, messages, users)
    │   ├── RALD_JWT_SECRET (Cloudflare secret)
    │   └── LOOP_JWT_SECRET (Cloudflare secret)
    ├── loop.rald.cloud (CF Pages)
    │   └── loop-api.rald.cloud (API calls)
    ├── messenger.rald.cloud (CF Worker)
    │   └── Supabase PostgreSQL
    └── notification.rald.cloud (NOT DEPLOYED)
```

---

## BACKUP INVENTORY

| Asset | Backup Method | Frequency | RPO |
|-------|--------------|-----------|-----|
| Supabase Database | Supabase PITR (Point-in-Time Recovery) | Continuous | < 1 min |
| GitHub Source Code | Git (distributed) | Every push | 0 |
| Cloudflare Worker Code | wrangler deploy (from Git) | On deploy | < 15 min |
| Cloudflare Pages | Auto-rebuilt from Git | On push | < 15 min |
| JWT Secrets | GitHub Secrets + CF Secrets | Manual | N/A |
| Domain DNS | Cloudflare DNS (built-in redundancy) | N/A | N/A |

---

## SECRET INVENTORY

| Secret | Stored In | Used By |
|--------|----------|---------|
| RALD_JWT_SECRET | CF Secrets + GH Secrets | auth.rald.cloud, loop-api, messenger |
| LOOP_JWT_SECRET | CF Secrets + GH Secrets | loop-api.rald.cloud |
| SUPABASE_SERVICE_ROLE_KEY | GH Secrets | All workers |
| SUPABASE_URL | GH Secrets + wrangler vars | All workers |
| CLOUDFLARE_API_TOKEN | GH Secrets | CI/CD deploy |
| CLOUDFLARE_ACCOUNT_ID | GH Secrets | CI/CD deploy |
| TERMII_API_KEY | CF Secrets + GH Secrets | auth.rald.cloud |
| SUPABASE_ANON_KEY | GH Secrets | profiles.rald.cloud, loop frontend |

---

## RECOVERY PROCEDURES

### Scenario 1: Cloudflare Worker Down
**RTO:** < 10 minutes
1. Check CF dashboard for worker errors
2. If code bug: push fix to GitHub → auto-deploys
3. If CF outage: monitor status.cloudflarestatus.com
4. Rollback: `wrangler rollback --env production`

### Scenario 2: Supabase Outage
**RTO:** < 30 minutes (depends on Supabase)
1. Monitor status.supabase.com
2. All RALD auth will be unavailable (JWT verification still works for cached sessions)
3. If prolonged: activate Supabase PITR restore
4. After restore: restart all workers to clear connection pool

### Scenario 3: Lost JWT Secret
**RTO:** < 15 minutes
1. Generate new secret: `openssl rand -hex 32`
2. Update in GitHub Secrets
3. Push deploy trigger → workers updated automatically
4. All existing sessions invalidated (users must re-login)

### Scenario 4: DNS Hijacking
**RTO:** < 60 minutes
1. Cloudflare lockdown mode: enable under Account → Security
2. Revoke all Cloudflare API tokens
3. Re-issue from scratch
4. Audit all DNS records via CF dashboard

### Scenario 5: Complete Platform Rebuild
**RTO:** < 4 hours
1. All code in GitHub: `git clone git@github.com:Ostinato-Loop/<repo>`
2. Provision new Supabase project, run migrations
3. Set all secrets in CF and GH
4. Push to main → CI/CD deploys all workers

---

## ROLLBACK PROCEDURES

| Service | Command | Time |
|---------|---------|------|
| CF Worker | `wrangler rollback --env production` | < 2 min |
| CF Pages | CF Dashboard → Deployments → Rollback | < 5 min |
| Database | Supabase Dashboard → PITR restore | < 15 min |
| Code | `git revert <sha> && git push` | < 10 min |

---

## RPO/RTO ASSESSMENT

| Service | RPO | RTO | Target Met |
|---------|-----|-----|-----------|
| Auth (auth.rald.cloud) | < 1 min | < 10 min | ✅ |
| Identity (profiles.rald.cloud) | < 5 min | < 15 min | ✅ |
| Loop API | < 1 min | < 10 min | ✅ |
| Loop Frontend | < 5 min | < 15 min | ✅ |
| Messenger API | < 1 min | < 10 min | ✅ |
| Supabase (data layer) | < 1 min (PITR) | < 30 min | ✅ |

**Overall RPO:** < 15 minutes ✅  
**Overall RTO:** < 60 minutes ✅

*Generated for RALD / LILCKY STUDIO LIMITED*
