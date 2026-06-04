# Cloudflare Service Bindings Migration Plan
> Generated: 2026-06-04 | Sprint: Foundation Stabilization — Task 4
> Documents current worker-to-worker HTTP calls and migration path to CF Service Bindings.

---

## What Are Service Bindings?

Cloudflare Service Bindings allow a Worker to call another Worker directly without:
- Going through the public internet
- DNS resolution / TLS overhead / egress costs

Calls stay within Cloudflare's network edge — faster, cheaper, more secure.

---

## Current State: Worker-to-Worker HTTP Calls

### rald-inbox → notification.rald.cloud
```toml
[vars]
NOTIFICATION_SERVICE_URL = "https://notification.rald.cloud"
```
Usage: inbox triggers notifications on SLA breach. Problem: public URL, DNS+TLS overhead.

### rald-inbox → search.rald.cloud
```toml
[vars]
SEARCH_SERVICE_URL = "https://search.rald.cloud"
```
Usage: inbox calls search to index/query conversations.

### loop-api → auth.rald.cloud
Usage: /api/auth/rald-sso validates RALD tokens via HTTP to auth.rald.cloud/sso/exchange.

### messenger → rald-realtime
Usage: Messenger calls realtime.rald.cloud for room/session management.

---

## Proposed Migration: Service Bindings

### Phase 1 — High Priority

**rald-inbox → rald-notify**
```toml
[[services]]
binding  = "NOTIFY"
service  = "rald-notify"
```

**rald-inbox → rald-search**
```toml
[[services]]
binding  = "SEARCH"
service  = "rald-search"
```

### Phase 2 — Medium Priority

**loop-api → rald-auth-core**
```toml
[[services]]
binding  = "RALD_AUTH"
service  = "rald-auth"
```

**messenger → rald-realtime**
```toml
[[services]]
binding  = "REALTIME"
service  = "rald-realtime"
```

---

## Migration Checklist (per binding)

- [ ] Identify all HTTP calls to target service in source worker
- [ ] Add `[[services]]` binding to source wrangler.toml
- [ ] Update code: replace `fetch("https://...")` with `env.BINDING.fetch(...)`
- [ ] Decide internal auth: shared INTERNAL_SECRET or remove JWT check for service calls
- [ ] Test in staging; verify no public URL fallback remains

---

## Blockers Before Migration

1. KV namespaces not provisioned — all RATE_LIMIT_KV IDs still `REPLACE_WITH_*`
2. No staging environment exists — all testing on production
3. Internal auth strategy undecided

---

## Status: DOCUMENTATION ONLY

Do not implement until: CI is green, KV provisioned, staging exists, internal auth decided.
