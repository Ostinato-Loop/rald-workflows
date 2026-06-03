# OBSERVABILITY CERTIFICATION
**Phase:** G.12 — Workstream 6
**Generated:** 2026-06-03 12:11 UTC

---

## CURRENT MONITORING COVERAGE

| Metric | Data Source | Available | Dashboard |
|--------|------------|-----------|-----------|
| Worker uptime | Cloudflare Analytics | ✅ | Cloudflare dashboard |
| Request count | CF Analytics | ✅ | Cloudflare dashboard |
| Error rate | CF Analytics | ✅ | Cloudflare dashboard |
| Auth Success % | Supabase logs | ✅ | Manual query |
| OTP Success % | Supabase auth_logs | ✅ | Manual query |
| Active sessions | Supabase sessions table | ✅ | Manual query |
| SSO Success % | Loop worker logs | ⚠ | CF Logs only |
| Loop Active Users | N/A | ❌ | Not implemented |
| Messenger Active Users | N/A | ❌ | Not implemented |
| Notification Deliveries | rald-notify | ⛔ | Not deployed |
| Search Latency | rald-search | ⛔ | Not deployed |
| Supabase Health | Supabase status page | ✅ | External |

---

## HEALTH ENDPOINTS

| Service | Endpoint | Response |
|---------|----------|----------|
| auth.rald.cloud | GET /health | `{"status":"ok","version":"2.1.0"}` |
| loop-api.rald.cloud | GET /api/health | `{"status":"healthy",...}` |
| messenger.rald.cloud | GET /health | `{"status":"ok"}` |
| profiles.rald.cloud | GET / | HTML 200 |

---

## OBSERVABILITY REQUIREMENTS (NOT YET BUILT)

### admin.rald.cloud Dashboard
**Status:** NOT DEPLOYED  
**Required Metrics:**
- Auth Success Rate (real-time, 5-minute rolling)
- OTP Delivery Rate
- SSO Exchange Success Rate
- Active Session Count
- Failed Login Attempts (with IP/device grouping)
- Worker Error Rate per service
- Supabase connection health
- Cloudflare D1/KV binding health

### Recommended Stack
- **Metrics:** Cloudflare Analytics Engine (zero-cost, built-in)
- **Alerting:** Cloudflare Notifications → webhook to Slack/email
- **Dashboard:** admin.rald.cloud Hono Worker + React frontend
- **Logs:** Cloudflare Logpush → R2 → queryable via Cloudflare Workers

---

## ACTION ITEMS

1. Enable Cloudflare Analytics Engine on all Workers (1-line wrangler.toml change)
2. Build admin.rald.cloud with real-time metrics dashboard
3. Configure Cloudflare alerting: error rate >5% → immediate notification
4. Add structured logging to all workers: `console.log(JSON.stringify({...}))`

---

## CERTIFICATION STATUS

**CERTIFIED FOR:** Internal monitoring via manual Supabase queries  
**NOT CERTIFIED FOR:** Production-grade automated alerting  
**Blocking:** admin.rald.cloud dashboard not built  

*Generated for RALD / LILCKY STUDIO LIMITED*
