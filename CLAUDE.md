# Link Tracker — Project Conventions

## Overview

Production Node.js link-tracking SaaS. Express 4.21, SQLite (better-sqlite3), vanilla JS + Tabler dashboard. Runs as Windows NSSM service. 16 background timers, zero automated tests.

## Orchestrator Entry Point

For multi-domain tasks (anything touching 2+ subdomains like backend + frontend, or deploy + monitoring), route through `@tech-lead`. It decomposes the request and delegates to the right specialist agents. Direct `@agent-name` invocation is for single-domain work only — e.g., `@test-agent` for writing tests, `@frontend-agent` for a UI change. When in doubt, use `@tech-lead`.

## Architecture

```
server/
  index.js          — Express app, middleware stack, 13 background jobs, WebSocket
  database.js       — SQLite schema (33 tables), WAL config, migrations
  config.js         — Dynamic config persisted to data/config.json
  middleware/
    auth.js         — JWT, API key auth, CSRF, permission checks
  routes/
    redirect.js     — Click tracking, bot detection, challenge pages, device targeting (1,433 lines)
    apilinks.js     — API campaign links CRUD + analytics (1,072 lines)
    analytics.js    — Dashboard analytics, read-only aggregations (820 lines)
    links.js        — Web link CRUD (678 lines)
    admin.js        — System config, user mgmt, domain verification (1,165 lines)
    domains.js      — Domain CRUD, DNS, SSL cert management (1,054 lines)
    + 8 smaller route files (auth, campaigns, advanced, webhooks, apikeys, teams, templates, qr)
  utils/
    helpers.js      — MEGA-UTILITY (1,649 lines): ID gen, slug gen, UA parsing, bot detection, IP utils, logging
    performance.js  — ClickBatcher (2s flush), AnalyticsCache (60s cleanup), LinkCache (30s eviction), StatsTracker
    ip-threat.js    — IP reputation scoring
    ip-datacenter.js — Datacenter IP detection, ASN lookup
  ssl/
    manager.js      — Let's Encrypt ACME automation (1,027 lines)

views/
  sectionsv3/       — Dashboard source files (assembled into views/v3/dashboard.html)
    inline.js       — All dashboard JavaScript (3,245 lines)
    assemble.js     — Build script: node views/sectionsv3/assemble.js --save

public/             — Static assets (Tabler, ApexCharts, JSVectorMap)
data/               — Runtime data (config.json, SQLite DB, SSL certs)
```

## Key Patterns

### Bot Detection
- 30+ signals scored in `helpers.js:computeBotScore()`
- Thresholds: trusted <15, suspicious 15-39, bot 40-69, block 70+
- Challenge page served to suspicious visitors (JS-based proof of browser)
- Honeypot links auto-learn bot profiles (bot_blueprints table)

### Database Migrations
- **Legacy pattern (24 migrations):** `try { db.prepare("SELECT col FROM table LIMIT 1").get(); } catch(e) { db.prepare("ALTER TABLE table ADD COLUMN col ...").run(); }`
- **New pattern (use this for all new migrations):** Schema version table
```js
const ver = db.prepare("SELECT version FROM schema_version").get();
if (ver.version < N) {
  db.prepare("ALTER TABLE ...").run();
  db.prepare("UPDATE schema_version SET version=?").run(N);
}
```
- Never use `unique_clicks` column on campaigns — only `total_clicks` exists
- Always test migrations against production DB copy (394MB) before deploying

### Missing Indexes (known gaps)
- clicks: (campaign_id, is_bot, clicked_at)
- clicks: (fraud_score, clicked_at)
- links: (user_id, link_type, is_honeypot)
- links: (recipient_email, link_type)
- comments: (entity_type, entity_id, created_at)

### Performance
- ClickBatcher buffers clicks in memory, flushes every 2s or at 2000 clicks
- LinkCache caches link lookups with 30s TTL
- AnalyticsCache caches dashboard queries with 2min TTL
- SQLite WAL mode with 256MB cache, 1GB mmap, 8192-byte pages
- WAL checkpoint every 5min (PASSIVE mode)

### Frontend Dashboard
- Vanilla JS (no framework) — all logic in views/sectionsv3/inline.js
- Uses Tabler (Bootstrap 5) components
- API wrapper: `API.get()`, `API.post()`, `API.put()`, `API.delete()`
- HTML escaping: always use `escHtml()` for user content
- Build: `node views/sectionsv3/assemble.js --save` after editing sectionsv3/ files

### Device Targeting
- Campaign-level `allowed_devices` field: all, mobile, desktop, tablet, mobile,tablet, desktop,tablet
- Wrong device → branded alert page with QR code (not a redirect)
- Clicks recorded with `challenge_status='device_blocked'`

### Deployment
- Windows Server + NSSM service
- Deploy script: Desktop/deploy-update.cmd (extracts ZIP, kills node, restarts)
- Backup created before each deploy in ../backups/
- Automatic rollback from backup ZIP if post-deploy health check fails
- No CI/CD — manual process

## Error Handling Conventions

### Response Shape
All API responses follow a strict, consistent pattern:

```js
// Success (GET/list)
res.json({ success: true, data: [...], pagination: { total, page, limit, pages } });

// Success (POST/create) — always 201
res.status(201).json({ success: true, campaign: { id, name, ... } });

// Success (PUT/DELETE)
res.json({ success: true });

// Error — always { error: "message" }, never { success: false }
res.status(400).json({ error: 'Human-readable error message' });
```

### HTTP Status Codes
| Code | Usage |
|------|-------|
| 200 | GET success, PUT/DELETE success |
| 201 | POST/create success (always, not 200) |
| 400 | Validation failure, malformed request |
| 401 | Missing/invalid/expired auth token or API key |
| 403 | Permission denied, CSRF failure, disabled account |
| 404 | Resource not found |
| 409 | Conflict (duplicate username, slug taken) |
| 429 | Rate limit exceeded |
| 500 | Internal error (catch block) |
| 503 | Server overload (load shedding), DB unavailable |

### Error Handling Pattern
Every route handler uses try-catch with early returns:

```js
router.post('/endpoint', requireAuth, (req, res) => {
  try {
    if (!req.body.name) return res.status(400).json({ error: 'Name required' });
    // business logic...
    res.status(201).json({ success: true, resource: {...} });
  } catch (err) {
    console.error('createResource:', err);
    res.status(500).json({ error: 'Failed to create resource' });
  }
});
```

- Routes do NOT use `next(err)` — all errors handled locally
- Global error handler (index.js:571) catches only URIError from bot scanners
- Catch blocks return generic messages (never expose internal errors to client)

## Security Conventions

### IP Anonymization
- Controlled by two config flags: `track_ip` (default: true) and `anonymize_ip` (default: false)
- When `anonymize_ip=true`: raw IP replaced with SHA256 hash via `hashIp(ip)`
- When `track_ip=false`: no IP stored at all
- Queries automatically switch between `ip_address` and `ip_hash` columns based on config

### Challenge Tokens (Bot Detection)
- Format: `<timestamp_base36>.<hmac_sha256_first16chars>`
- HMAC input: `timestamp:linkId:visitorIP` signed with `_challengeSecret` (persisted in config.json)
- Valid window: **800ms to 15 seconds** — too fast = automated, too old = replay attack
- Delivered as URL query param `_bv=`, not cookies

### Rate Limiting
| Limiter | Window | Max | Applied To |
|---------|--------|-----|------------|
| General | 15 min | 100 req | `/auth` routes |
| API | 1 min | 300 req | `/api/*` routes |
| Password reset | 15 min | 5 req | `/auth/forgot-password` |
| Bulk operations | 1 min | 10 req | `/api/links/bulk` |
| Challenge serve | 5 min | 20 per IP | redirect.js bot challenges |
| Login brute-force | 15 min | 10 failures | per IP, blocks 15 min |
| Account lockout | 1 hour | 20 failures | per account, locks 1 hour |
| Per-API-key | 1 min | configurable | set per key in api_keys table |

### CSRF Protection
- Cookie: `_csrf` (httpOnly, secure if SSL, sameSite=strict, 24h expiry)
- Header: `X-CSRF-Token` (client sends, server compares to cookie)
- Skipped for: API key auth, GET/HEAD/OPTIONS, same-origin requests (Origin/Referer matches Host)
- Applied to: all `/api/*` routes

### JWT Authentication
- Default expiry: 7 days; with `remember_me`: 30 days
- Cookie: `token` (httpOnly, secure if SSL, sameSite=lax)
- Invalidated on password change (compares `iat` vs `password_changed_at`)
- Used for: dashboard/web UI access

### API Key Authentication
- Header: `X-API-Key`
- Storage: SHA256 hash of raw key in `api_keys` table
- Write permission: checked via `requireWritePermission` middleware (only for POST/PUT/DELETE)
- CSRF not required for API key auth
- Used for: programmatic API access, external integrations

## Anti-Patterns to Avoid

1. **Don't add to helpers.js** — it's already 1,649 lines. Create new files in server/utils/ instead
2. **Don't use raw SQL without db.prepare()** — always use parameterized queries with `?` placeholders
3. **Don't assume unique_clicks exists on campaigns** — it doesn't; only total_clicks
4. **Don't add console.log without context** — include function name and relevant IDs
5. **Don't modify redirect.js click recording without updating analytics queries** — they must agree on column names and challenge_status values
6. **Don't skip the assemble step** — editing sectionsv3/ files without running assemble.js means changes won't appear
7. **Don't add npm dependencies without checking** — production has only 13 dependencies; keep it minimal
8. **Don't create migration scripts in server/utils/** — put migrations in database.js using the versioned pattern
9. **Don't run WAL checkpoint with FULL or RESTART mode during traffic** — only PASSIVE is safe under load. FULL/RESTART block writers and can cause request timeouts. The 5-minute checkpoint job correctly uses PASSIVE; never change it.

## Testing Priority

1. Bot scoring (computeBotScore) — highest business risk, zero tests
2. Analytics SQL queries — complex aggregations, easy to break
3. Challenge flow (redirect.js) — security-critical, token validation
4. Click deduplication — business logic correctness
5. API campaign CRUD — data integrity

## Background Jobs (16 timers total)

13 in index.js + 3 in performance.js. All run via setInterval with silent try-catch. No health monitoring exists. See monitoring-agent for planned improvements.

### index.js timers (13)

| Job | Interval | Line |
|-----|----------|------|
| WS rate limit reset | 10s | index.js:628 |
| WS ping keepalive | 30s | index.js:641 |
| Click rate cleanup | 30min | index.js:764 |
| Daily scheduler check | 1h | index.js:1045 |
| Anomaly detection | 10min | index.js:1059 |
| Fraud learning | 5min | index.js:1061 |
| WAL checkpoint (PASSIVE) | 5min | index.js:1065 |
| Honeypot flag cleanup | 1h | index.js:1070 |
| Challenge timeout handling | 5min | index.js:1086 |
| Blueprint confidence decay | 7 days | index.js:1110 |
| Retroactive bot detection | 2min | index.js:1141 |
| SSL renewal check | 24h | index.js:1159 |
| Memory watchdog | 1min | index.js:1186 |

### performance.js timers (3)

| Job | Interval | Line |
|-----|----------|------|
| ClickBatcher flush | 2s | performance.js:53 |
| AnalyticsCache TTL cleanup | 60s | performance.js:158 |
| LinkCache expiry eviction | 30s | performance.js:284 |
