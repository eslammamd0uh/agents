# Subagents Strategy Report

## Link Tracker Project

Comprehensive analysis of AI subagent architecture: identifying operational needs, evaluating existing GitHub agents, and designing a custom orchestration system tailored to a production link-tracking SaaS platform.

**Generated:** April 13, 2026
**Analysis by:** Claude Code (Opus 4.6)
**Classification:** Confidential

---

# Table of Contents

| # | Section | Page |
|---|---------|------|
| 1 | Executive Summary | 3 |
| 2 | Project Profile | 4 |
| 3 | Methodology | 6 |
| 4 | Identified Needs | 7 |
| 5 | Main Agents — Detailed Comparison | 9 |
| 6 | Orchestrator Analysis | 16 |
| 7 | Final Recommendations Table | 20 |
| 8 | Custom Agent Full Drafts | 21 |
| 9 | Installation Plan | 30 |
| 10 | Usage Guide | 32 |
| 11 | Self-Review Summary | 34 |
| 12 | Decisions Applied | 36 |
| 13 | Appendix A: All GitHub Agents Considered | 37 |
| 14 | Appendix B: Repository References | 39 |
| 15 | Appendix C: Glossary | 40 |

---

# 1. Executive Summary

**Link Tracker** is a production Node.js link-tracking SaaS (Express 4.21, SQLite, Tabler dashboard) with 33 database tables, 15 route files, 30+ bot detection signals, and 220+ custom domains under SSL management. The platform handles campaign-based link generation, real-time click analytics, device/geo targeting, conversion tracking, and webhook integrations — all running as a Windows NSSM service with zero automated tests and no CI/CD pipeline.

> **Key Finding:** The project's greatest risk is its complete lack of testing (0 tests for 13K+ lines of server code), followed by invisible background job failures (16 setInterval timers with no monitoring), and manual deployment via batch scripts.

## Recommendation Summary

| Category | Count | Details |
|----------|-------|---------|
| Custom Agents | 6 | tech-lead (orchestrator), test-agent, api-doc-agent, deploy-agent, monitoring-agent, frontend-agent |
| Ready-made Agents | 3 | security-auditor (iannuttall), performance-engineer (lst97), backend-architect (lst97) |
| **Total** | **9** | **6 custom + 3 ready-made** |

## Top 3 Highest-Impact Recommendations

- **1. Custom test-agent** (Critical) — Zero tests for complex bot scoring, challenge flows, and analytics SQL. Every deploy is a gamble.
- **2. Custom deploy-agent + monitoring-agent** (High) — Windows + NSSM + batch scripts. No existing agent handles this unusual stack. Deploy handles ZIP/rollback; monitoring split out to dedicated agent for 16 background timers.
- **3. Custom tech-lead orchestrator** (High) — Single entry point that routes requests to the right specialist, preventing context fragmentation.

**Estimated implementation effort:** 2-3 hours to create all custom agent prompts and install ready-made agents. No code changes required — agents are markdown prompt files placed in `.claude/agents/`.

---

# 2. Project Profile

## 2.1 Tech Stack

| Component | Technology | Details |
|-----------|-----------|---------|
| Runtime | Node.js 18+ | v24.14.0 in production |
| Framework | Express 4.21.0 | 13 dependencies total |
| Database | SQLite (better-sqlite3) | WAL mode, 394MB production DB |
| Frontend | Vanilla JS + Tabler | Bootstrap 5, ApexCharts, JSVectorMap |
| Auth | JWT + API Keys | bcrypt-12, HTTP-only cookies, CSRF |
| SSL | Let's Encrypt (ACME) | 220+ custom domains, auto-renewal |
| Deployment | Windows Server + NSSM | Manual CMD script, no CI/CD |
| Real-time | WebSocket (ws) | Live click feed to dashboard |
| Testing | None | Zero test files, no test framework |

## 2.2 Architecture Overview

Monolithic Node.js application serving the dashboard, REST API, and link redirects from a single Express server. SQLite with advanced WAL tuning (256MB cache, 1GB mmap, 8192-byte pages) provides the data layer. No microservices, no message queue, no external cache — all caching is in-process (LinkCache, AnalyticsCache, ClickBatcher).

## 2.3 Key Modules

| Module | File | Lines | Purpose |
|--------|------|-------|---------|
| Redirect Engine | redirect.js | 1,433 | Click tracking, bot challenge, device/geo/A-B routing |
| Bot Scoring | helpers.js | 1,649 | 30+ detection signals, honeypot learning, UA parsing |
| Campaign API | apilinks.js | 1,072 | Bulk generation (100K), analytics, device targeting |
| Dashboard JS | inline.js | 3,245 | All frontend logic, API calls, charts, modals |
| Admin Panel | admin.js | 1,165 | Domain setup, SSL, user management, system stats |
| Schema | database.js | 921 | 33 tables, 50+ indexes, migrations |
| SSL Manager | ssl/manager.js | ~1,200 | ACME, cert renewal, self-signed fallback |
| Performance | performance.js | 344 | ClickBatcher, LinkCache, AnalyticsCache |

## 2.4 Background Jobs

The application runs **16 background timers (13 in index.js + 3 in performance.js)** via native setInterval/setTimeout (no external job queue):

| Job | Interval | Location | Purpose |
|-----|----------|----------|---------|
| ClickBatcher flush | 2s | performance.js:53 | Batch-writes buffered clicks to SQLite |
| LinkCache eviction | 30s | performance.js:284 | Removes stale cache entries (10s TTL) |
| AnalyticsCache TTL cleanup | 60s | performance.js:158 | Evicts entries older than 2 minutes |
| WS rate limit reset | 10s | index.js:628 | Resets WebSocket rate limit counters |
| WS ping keepalive | 30s | index.js:641 | Pings connected dashboard clients |
| Anomaly detection | 10min | index.js:700 | Detects click anomalies per link |
| Fraud learning | 5min | index.js:730 | Updates fraud pattern database |
| Click rate cleanup | 30min | index.js:764 | Prunes stale click rate tracking data |
| Memory watchdog | 60s | index.js:800 | Prunes caches if RSS > 1GB |
| Daily scheduler | 1h | index.js:1045 | Purges old logs, expired tokens |
| Honeypot flag cleanup | 1h | index.js:1070 | Removes stale honeypot flags |
| Challenge timeout | 5min | index.js:1086 | Expires pending challenge sessions |
| Blueprint confidence decay | 7 days | index.js:1110 | Decays bot blueprint confidence scores |
| Retroactive bot check | 2min | index.js:1141 | Re-evaluates recent clicks with new intel |
| SSL renewal | 24h | index.js:1159 | Renews expiring Let's Encrypt certificates |
| Daily cleanup | 24h | index.js:1045 | Purges old logs, expired tokens |

## 2.5 Pain Points Observed

- **Zero test coverage** — 13K+ lines of server code with no automated tests
- **No logging framework** — console.log with prefixes, no structured logging
- **No error tracking** — no Sentry, Bugsnag, or monitoring dashboard
- **Manual deployment** — CMD batch script with manual ZIP transfer between RDP sessions
- **Synchronous webhooks** — webhook delivery blocks the redirect flow (5s timeout)
- **Legacy migrations** — 24 try/catch ALTER TABLE migrations coexisting with 4 versioned migrations
- **Monolithic frontend** — 264KB single-file inline.js with race conditions on view switching

---

# 3. Methodology

## 3.1 Codebase Analysis

The analysis was conducted in three passes:

- **Pass 1 — Structure scan:** Read package.json, directory listings, database schema, all route files, utility modules, middleware, and configuration files.
- **Pass 2 — Deep audit:** Re-read every key file with line-level verification. Counted tables (33), route files (15), background timers (16). Verified dependency list, caching mechanisms, error handling patterns.
- **Pass 3 — Self-review:** Cross-checked every claim against the actual codebase. Found and corrected 13 issues (1 critical, 8 important, 4 minor).

## 3.2 GitHub Repositories Researched

| Repository | Stars | Agents | Format |
|-----------|-------|--------|--------|
| wshobson/agents | 33,500 | 182 | Plugin system (multi-file) |
| VoltAgent/awesome-claude-code-subagents | 17,100 | 130+ | .md files + plugin support |
| 0xfurai/claude-code-subagents | 827 | 139 | Flat .md files |
| iannuttall/claude-agents | 2,000 | 7 | Flat .md files (curated) |
| lst97/claude-code-sub-agents | 1,500 | 37 | Categorized .md files |
| rahulvrane/awesome-claude-agents | 305 | N/A | Curated directory (links) |
| vijaythecoder/awesome-claude-agents | N/A | ~15 | Categorized .md files |

## 3.3 Evaluation Criteria

- **Domain relevance:** Does the agent understand Node.js/Express/SQLite/vanilla JS?
- **Tool permissions:** Does it use the right tools (Agent for delegation, Read/Edit for code)?
- **Model quality:** Opus > Sonnet > Haiku for complex tasks (haiku flagged as concern)
- **Delegation capability:** Can it actually spawn subagents (Agent tool) or just plan?
- **Customization cost:** How much modification needed for link-tracker specifically?

---

# 4. Identified Needs

### Need #1: Testing (Unit + Integration)

**Priority:** Critical

Zero test coverage for 13K+ lines including a 30-signal bot scoring engine, challenge state machine, analytics SQL, and click attribution logic.

**Evidence:** `helpers.js (computeBotScore), redirect.js (challenge flow), apilinks.js (analytics SQL), database.js (schema)`

### Need #2: Security Auditing

**Priority:** High

JWT auth, API keys, redirect URL handling, CSP policy, and IP anonymization need formal OWASP review. Custom domains with SSL add attack surface.

**Evidence:** `middleware/auth.js, redirect.js (URL injection surface), admin.js (domain/SSL management)`

### Need #3: API Documentation

**Priority:** High

60+ endpoints across 15 route files. AI agents consume the docs — must be accurate and in specific format. Manual updates miss new features.

**Evidence:** `apilinks.js, links.js, analytics.js, admin.js, campaigns.js, webhooks.js`

### Need #4: Deployment Automation

**Priority:** High

Manual CMD script with ZIP transfer between RDP sessions. Windows Server + NSSM + PowerShell verification. No CI/CD, no rollback.

**Evidence:** `deploy-update.cmd, install-service.js, NSSM configuration`

### Need #5: Architectural Planning

**Priority:** Medium

Complex monolith with growing features. Needs guidance on API design, schema decisions, and scalability planning for new features.

**Evidence:** `database.js (33 tables), apilinks.js (campaign architecture), redirect.js (routing complexity)`

### Need #6: Performance Optimization

**Priority:** Medium

394MB SQLite DB, 50+ indexes, 3 caching layers, synchronous webhook delivery. Needs profiling and bottleneck analysis.

**Evidence:** `performance.js (caching), database.js (indexes), redirect.js (webhook sync calls)`

### Need #7: Frontend Dashboard

**Priority:** Medium

264KB vanilla JS file, Tabler Bootstrap 5, assemble.js build pipeline. Race conditions on view switching. WebSocket live feed.

**Evidence:** `inline.js, modals.html, layout.html, assemble.js, sectionsv3/ templates`

### Need #8: Monitoring & Observability

**Priority:** Medium

16 background timers on setInterval with no visibility. Memory watchdog silently prunes. No structured logging. No error tracking.

**Evidence:** `index.js (all setInterval jobs), performance.js (ClickBatcher, caches)`

---

# 5. Main Agents — Detailed Comparison

## 5.1 Need #1: Testing (Unit + Integration)

### Option A — Ready-made: test-automator (lst97)

- Understands unit/integration/e2e patterns, can scaffold Jest config
- **Con:** No knowledge of bot scoring signals, challenge state machine, or click-batching logic
- **Con:** Will generate generic Express route tests missing actual edge cases
- Customization needed: Heavy — must teach thresholds, states, SQL patterns

### Option B — Build Custom

- Encodes exact domain logic: score ranges (0-14/15-39/40-69/70+), challenge states, click dedup
- Tests against actual SQLite schema with in-memory DB
- Covers bot scoring, analytics SQL, post-challenge logic, device targeting
- Complexity: **Medium**

> **Recommendation: Custom** — Domain-specific test needs are too unique for any generic test agent.

## 5.2 Need #2: Security Auditing

### Option A — Ready-made: security-auditor (iannuttall/claude-agents)

- Comprehensive OWASP Top 10 coverage, 9 vulnerability categories
- Has write tools (Edit, Bash) — can find AND fix issues
- Produces structured security-report.md with severity ratings
- **Con:** Model not specified (inherits parent — Opus in your case, which is good)
- **Con:** Generic — doesn't know your redirect URL injection surface or ACME flow

### Option B — Build Custom

- Focuses on redirect URL injection, API key scoping, config.json secrets, challenge page XSS
- Complexity: **Medium-High**

> **Recommendation: Ready-made (iannuttall)** — Security checklists are universal. Supplement with domain-specific notes in the prompt.

## 5.3 Need #3: API Documentation

### Option A — Ready-made: api-documenter (lst97)

- Can generate OpenAPI/Swagger specs
- **Con:** Doesn't know your route structure, response format, or the markdown format your AI agents consume
- Customization needed: High

### Option B — Build Custom

- Reads your route files, extracts endpoints, generates docs in your exact format
- Auto-detects new endpoints when routes change
- Outputs markdown consumable by your campaign automation AI agents
- Complexity: **Low**

> **Recommendation: Custom** — Your AI agents need docs in a specific format. No existing agent generates it.

## 5.4 Need #4: Deployment Automation

### Option A — Ready-made: deployment-engineer (VoltAgent)

- Covers CI/CD pipelines, Docker, cloud deployment
- **Con:** Assumes Linux/cloud/containers. Your stack is Windows Server + NSSM + batch scripts
- Customization needed: Essentially a complete rewrite

### Option B — Build Custom

- Knows Windows, NSSM, PowerShell verification, ZIP with System.IO.Compression, flat-file cleanup
- Includes automatic rollback from backup ZIP
- Monitoring split out to dedicated monitoring-agent (see Need #8)
- Complexity: **Low**

> **Recommendation: Custom** — No existing agent handles Windows + NSSM. Low complexity, high value.

## 5.5 Need #5: Architectural Planning

### Option A — Ready-made: backend-architect (lst97)

- Model: Sonnet — capable for architectural guidance
- Specializes in API design, schema planning, scalability, security patterns
- Better fit than code-reviewer (Haiku) for planning new features

### Option B — Build Custom

- Could encode your specific patterns (SQLite constraints, Express middleware chain)
- **Con:** Architecture benefits from fresh outside perspective
- Complexity: **Medium**

> **Recommendation: Ready-made (lst97 backend-architect)** — Architectural guidance benefits from broad knowledge, not project-specific customization.

> **Why not split into tracking-engine, analytics, API, database agents?** See Section 5.9 for full coupling analysis. Short answer: helpers.js (1,649 lines) is imported by every subdomain, database.js has no abstraction layer (90+ inline SQL), and real features routinely span 3-4 subdomains. One backend agent with domain-specific CLAUDE.md context outperforms multiple coordinating specialists.

## 5.6 Need #6: Performance Optimization

### Option A — Ready-made: performance-engineer (lst97)

- Model: Sonnet — covers profiling, caching, bottleneck analysis
- Full-stack: frontend Core Web Vitals, backend profiling, DB optimization
- **Con:** May not understand SQLite-specific PRAGMA tuning or WAL checkpointing

### Option B — Build Custom

- Knows your PRAGMA config, ClickBatcher, caching layers
- **Con:** You've already done most obvious optimizations
- Complexity: **Medium**

> **Recommendation: Ready-made (lst97 performance-engineer)** — Fresh perspective on things you haven't thought of.

## 5.7 Need #7: Frontend Dashboard

### Option A — Ready-made: frontend-developer (lst97)

- Understands responsive layouts, CSS, accessibility
- **Con:** Likely React/Vue-focused. Your dashboard is vanilla JS + Tabler
- Customization needed: High

### Option B — Build Custom

- Knows Tabler components, escHtml(), API.get/put/post/delete wrapper, modal system
- Knows assemble.js pipeline (sectionsv3/ source files compiled to dashboard.html)
- Scope includes WebSocket live feed functionality
- Complexity: **Medium**

> **Recommendation: Custom** — Vanilla JS + Tabler is unusual. Generic frontend agents will fight against your architecture.

## 5.8 Need #8: Monitoring & Observability

### Option A — Fold into deploy-agent (rejected)

- Deploy-agent focuses on ZIP packaging, NSSM stop/start, and rollback
- Monitoring is a continuous concern; deploy is a discrete event
- Monitoring requires read-only access; deploy requires write access
- Combining them violates single-responsibility and bloats the deploy prompt

### Option B — Dedicated monitoring-agent (selected)

- 16 background timers across index.js and performance.js with zero failure detection
- No structured logging framework -- all console.log with prefixes
- /health endpoint exists but is unused by any automated system
- Read-only agent: inspects state, never modifies code or config
- Natural fit for Sonnet model -- pattern recognition over code generation

### Monitoring-agent scope

- Background timer inventory and health verification
- Memory/cache monitoring (RSS, ClickBatcher, LinkCache, AnalyticsCache)
- Error pattern detection in console output
- /health endpoint utilization and alerting recommendations

> **Recommendation: Dedicated monitoring-agent** — Zero coupling with deploy-agent. Monitoring is continuous and read-only; deployment is discrete and write-heavy.

---

## 5.9 Backend Split Analysis: One Agent or Multiple?

A natural question arises: should the backend be served by one agent (backend-architect) or split into multiple specialists (tracking-engine, analytics, API-layer, database)? We analyzed four candidate subdomains against three criteria: coupling, independent change frequency, and unique complexity.

| Subdomain | Primary File | Lines | DB Tables R/W | Coupling |
|-----------|-------------|-------|---------------|----------|
| Tracking Engine | redirect.js | 1,433 | 10R / 5W | VERY TIGHT |
| Analytics | analytics.js | 820 | 3R / 0W | LOOSE (read-only) |
| API/CRUD Layer | 12 route files | ~3,900 | varies | MODERATE-HIGH |
| Database | database.js | 921 | all | FOUNDATION |

### Three Coupling Bottlenecks Prevent Clean Splits

**1. helpers.js (1,649 lines)** — A mega-utility imported by every subdomain. Contains 10+ functional domains: ID generation (12 routes), bot detection (redirect.js only, 300+ lines), URL building (links.js, apilinks.js), activity logging (7 routes), IP utilities (5 routes). Any agent touching bot detection risks breaking link creation, and vice versa. No module boundary exists.

**2. database.js (no abstraction)** — Every route file writes raw `db.prepare('SELECT ...')` inline. There are 90+ hand-written SQL statements scattered across routes with no query builder, no repository pattern, no shared statement pool. A schema change to `clicks` affects redirect.js (50+ statements), analytics.js (40+ statements), and apilinks.js simultaneously.

**3. performance.js bridges tracking and analytics** — ClickBatcher (redirect.js only) and AnalyticsCache (analytics.js + apilinks.js) live in the same file, share the same `init(db, config)` call, and are initialized together in index.js.

### Cross-Domain Feature Evidence

Real tasks routinely span 3-4 subdomains. Example: "Add allowed_devices to campaigns with device blocking" touched database.js (schema + migration), apilinks.js (POST/PUT accept field), redirect.js (device check before redirect), and analytics queries (device_blocked counter) — 4 files across 3 subdomains in one coherent change. Split agents would require orchestrator coordination for most non-trivial features.

### Verdict

> **Keep ONE backend agent.** The codebase has high cohesion with tight coupling. The subdomains are logically distinct but physically entangled through three shared bottlenecks. Splitting into multiple agents would create constant coordination overhead that exceeds the specialization benefit. The backend-architect (ready-made, lst97) with domain-specific CLAUDE.md guidance provides specialist-level context without the coordination tax.

**What would justify splitting later:** (1) helpers.js refactored into 5+ focused modules, (2) a query builder or repository pattern added, (3) codebase grows past 20K+ lines with clear domain boundaries, (4) analytics becomes a separate service (e.g., ClickHouse).

## 5.10 Logging & Error Tracking Strategy

### Current State

- All logging via `console.log` with ad-hoc prefixes (e.g., `[SSL]`, `[WS]`, `[BOT]`)
- No log levels, no structured output, no file rotation
- Errors caught but silently swallowed in many setInterval callbacks
- No error tracking service (no Sentry, Bugsnag, or equivalent)

### Recommended: Pino + File-Based Logging

- **Pino** — Fastest Node.js structured logger, JSON output, minimal overhead
- File-based transport (pino-file or pino/transport) — no external service dependency
- Log rotation via pino-roll or OS-level logrotate
- Retains Windows Server compatibility (no Unix syslog dependency)

### Implementation Phases

| Phase | Scope | Owner |
|-------|-------|-------|
| Phase 1 | Replace console.log in index.js with Pino logger, add log levels | backend-architect |
| Phase 2 | Add structured error context to all 16 background timers | monitoring-agent (audit) |
| Phase 3 | Add request-id tracing to redirect.js and API routes | backend-architect |

### Agent Ownership

- **monitoring-agent** audits logging gaps and recommends improvements
- **backend-architect** implements the Pino integration
- **deploy-agent** ensures log files are excluded from deploy ZIP and rotated

---

# 6. Orchestrator Analysis

## 6.1 Existing Orchestrators Evaluated

| Agent | Repo | Delegates? | Agent Tool? | Pattern |
|-------|------|-----------|-------------|---------|
| team-lead | wshobson | YES | YES | Master-worker, file ownership |
| tech-lead-orchestrator | vijaythecoder | Indirect | No | Outputs @agent assignments |
| agent-organizer | lst97 | No | No | Project analysis + team rec |
| multi-agent-coordinator | VoltAgent | No | No | Theoretical workflow design |
| workflow-orchestrator | VoltAgent | No | No | State machine design |
| task-distributor | VoltAgent | No | No | Queue/load balancing theory |

> **Key Finding:** Only ONE agent (wshobson's team-lead) actually delegates via the Agent tool. All VoltAgent meta-orchestration agents (6 total) are plan-only with no Agent tool. The lst97 agent-organizer is also plan-only. Most "orchestrators" in the ecosystem are theoretical, not operational.

## 6.2 Side-by-Side Comparison

### Option A — wshobson's team-lead

- **Pro:** Only agent that actually spawns subagents via Agent tool
- **Pro:** File ownership model prevents merge conflicts
- **Con:** Designed as a coding team lead, not a request router
- **Con:** No knowledge of your tech stack, subagent roster, or domain
- **Con:** Requires TeamCreate tool which may not be available
- **Con:** Over-engineered for your needs (5 parallel coding agents vs 1 dispatcher)

### Option B — Custom tech-lead

- **Pro:** Knows your exact 9-agent roster by name
- **Pro:** Domain-specific routing ("bot scoring" goes to test-agent, "SQL" goes to sqlite knowledge, etc.)
- **Pro:** Knows your file layout, deploy workflow, assemble.js pipeline
- **Pro:** Lightweight — just routing + aggregation, no unnecessary team management
- **Con:** Initial setup time (~30 min)
- **Con:** Needs updating when roster changes

> **Recommendation: Custom** — The gap between existing orchestrators and your needs is too large. A custom dispatcher tailored to your 9 agents and domain vocabulary saves hours of fighting a misfit generic agent.

## 6.3 Integration Architecture

```
                              User Request
                                   |
                                   v
                         +------------------+
                         |   tech-lead      |  <-- Orchestrator (Custom, Opus)
                         |  (dispatcher)    |
                         +--------+---------+
                                  |
                 Analyzes request, decomposes, routes
                                  |
     +-------+-------+------+----+----+--------+-------+--------+--------+
     |       |       |      |        |        |       |        |        |
     v       v       v      v        v        v       v        v        v
  +------+ +-----+ +----+ +-----+ +------+ +-----+ +------+ +------+ +-----+
  | test | | sec | |arch| | api | | perf | |front| |deploy| |monit | |sqlit|
  | agent| |audit| |itct| | doc | | engr | | end | |agent | |agent | |expr |
  +------+ +-----+ +----+ +-----+ +------+ +-----+ +------+ +------+ +-----+
  Custom   Ready   Ready  Custom   Ready   Custom  Custom   Custom   Ready
           iann.   lst97          lst97            rollback  read-    0xfurai
                                                             only
```

*Figure 1: Agent orchestration architecture*

---

# 7. Final Recommendations Table

| # | Agent Name | Type | Source | Priority | Purpose |
|---|-----------|------|--------|----------|---------|
| 1 | tech-lead | Custom | Build from scratch | **High** | Orchestrator: routes requests to specialists, ambiguity fallback |
| 2 | test-agent | Custom | Build from scratch | **Critical** | Bot scoring first, then challenge flow, analytics SQL tests |
| 3 | security-auditor | Ready-made | iannuttall/claude-agents | **High** | OWASP audit, auth review, vulnerability scan (model: sonnet) |
| 4 | api-doc-agent | Custom | Build from scratch | **High** | Generate/update API docs for AI agent consumption |
| 5 | deploy-agent | Custom | Build from scratch | **High** | Windows/NSSM deploy, ZIP packaging, automatic rollback |
| 6 | monitoring-agent | Custom | Build from scratch | **High** | 16 background timers, health checks, logging audit |
| 7 | backend-architect | Ready-made | lst97/claude-code-sub-agents | **Medium** | API design, schema planning, architecture + migrations |
| 8 | performance-engineer | Ready-made | lst97/claude-code-sub-agents | **Medium** | Profiling, caching, bottleneck analysis |
| 9 | frontend-agent | Custom | Build from scratch | **Medium** | Tabler UI, inline.js, modals, WebSocket |

> **Summary:** 6 custom agents + 3 ready-made agents = 9 total. Custom agents handle domain-specific needs (testing, docs, deploy, monitoring, frontend, orchestration). Ready-made agents handle universal concerns (security, architecture, performance).

---

# 8. Custom Agent Full Drafts

Each draft below is ready to copy into `.claude/agents/` as a `.md` file.

## 8.1 tech-lead.md (Orchestrator)

**File:** `.claude/agents/tech-lead.md`
**Invocation:** `@tech-lead` or auto-detected for multi-domain requests
**Tools:** Agent, Read, Glob, Grep, Bash, TodoWrite

```
---
name: tech-lead
description: Orchestrator for link-tracker. Receives requests, decomposes
  into subtasks, delegates to specialist subagents, aggregates results.
  Never implements directly.
model: opus
tools: Agent, Read, Glob, Grep, Bash, TodoWrite
---

# Link Tracker -- Tech Lead Orchestrator

You are the tech lead for the link-tracker project -- a Node.js
link-tracking SaaS with bot detection, analytics, campaigns,
and custom domains.

## Your Role

You are a DISPATCHER and COORDINATOR, NOT an implementer.
1. Analyze what's being asked
2. Decompose into subtasks
3. Route each subtask to the right specialist subagent
4. Aggregate results and report back
5. Never write code yourself -- always delegate

## Project Context

- Stack: Node.js 18+, Express 4.21, SQLite (better-sqlite3), vanilla JS
- Dashboard: Tabler (Bootstrap 5), assembled via
  views/sectionsv3/assemble.js -> views/v3/dashboard.html
- Database: 33 tables, SQLite WAL mode, server/database.js
- Bot Detection: 30+ signals in server/utils/helpers.js
  Thresholds: trusted <15, suspicious 15-39, bot 40-69, block 70+
- Deploy: Windows Server, NSSM service, deploy-update.cmd, ZIP
- Background: 16 setInterval timers (13 in index.js, 3 in performance.js)
- Conventions: See CLAUDE.md

## Subagent Roster

| Agent              | Type       | Use When                          |
|--------------------|------------|-----------------------------------|
| test-agent         | Custom     | Testing bot scoring, challenge    |
|                    |            | flow, analytics SQL, clicks       |
| security-auditor   | Ready-made | OWASP review, auth audit, CVEs   |
| backend-architect  | Ready-made | API design, schema planning,      |
|                    |            | migrations                        |
| api-doc-agent      | Custom     | Update API docs after changes     |
| performance-engr   | Ready-made | Profiling, caching, bottlenecks   |
| deploy-agent       | Custom     | Deploy ZIP, NSSM, rollback        |
| monitoring-agent   | Custom     | 16 timers, health, logging audit  |
| frontend-agent     | Custom     | inline.js, Tabler, modals, WS     |

## Routing Rules

Bot/Scoring      -> test-agent (testing), direct (code change)
Database/Schema  -> backend-architect (design), test-agent (testing)
API Endpoints    -> api-doc-agent (after), test-agent (testing)
Frontend/UI      -> frontend-agent
Security         -> security-auditor
Performance      -> performance-engineer
Deployment       -> deploy-agent
Monitoring       -> monitoring-agent
Architecture     -> backend-architect
Ambiguous        -> ask user to clarify before routing

## Workflow: Post-Change Pipeline

After ANY code change, suggest:
1. test-agent: Test the changes
2. api-doc-agent: Update docs if endpoints changed
3. Rebuild: node views/sectionsv3/assemble.js --save
4. deploy-agent: Update zip and deploy verification

## Rules

1. Never implement directly -- always delegate
2. Use TodoWrite to track subtask progress
3. Run subagents in parallel when tasks are independent
4. Always read relevant files first for accurate context
5. Aggregate clearly into a single coherent response
6. When request is ambiguous, ask user to clarify before routing
7. Route monitoring/health/timer questions to monitoring-agent
```

## 8.2 test-agent.md

**File:** `.claude/agents/test-agent.md`
**Tools:** Read, Write, Edit, Glob, Grep, Bash

```
---
name: test-agent
description: Testing specialist for link-tracker. Writes and runs tests
  for bot scoring, challenge flow, analytics SQL, click counting, and
  device targeting. Knows all thresholds and state transitions.
model: sonnet
tools: Read, Write, Edit, Glob, Grep, Bash
---

# Link Tracker -- Test Agent

You write and run tests for the link-tracker project.

## Tech Stack
- Runtime: Node.js 18+, Express 4.21
- Database: SQLite (better-sqlite3) -- use in-memory DB for tests
- Test framework: Jest (install if not present)
- Key files:
  - server/utils/helpers.js -- computeBotScore(), parseUserAgent()
  - server/routes/redirect.js -- challenge flow, device targeting
  - server/routes/apilinks.js -- analytics SQL, campaign CRUD
  - server/database.js -- schema, 33 tables

## Domain Knowledge

### Bot Scoring Thresholds
- 0-14: Trusted (auto-redirect, no challenge)
- 15-39: Suspicious (mouse challenge required)
- 40-69: Bot (challenge, blocked if still >=40 after)
- 70+: Instant block (no challenge)

### Challenge State Machine
- null -> passed (score <40 after challenge = human)
- null -> blocked (score >=40 after challenge = bot)
- null -> expired (challenge timeout)
- null -> scanner_blocked (email scanner detected)
- null -> device_blocked (wrong device type)

### Click Counting Rules
- Human = is_bot=0 (simple, no complex conditions)
- Passed challenge + score <40 = human (is_bot=0)
- Passed challenge + score >=40 = bot (is_bot=1)
- device_blocked clicks: is_bot=0, challenge_status=device_blocked

### Mouse Entropy Scoring (only when bot_score >= 15)
- No movement: +10 points
- Robotic pattern (pts>=5, var<0.001, angles=0): +8
- Natural (pts>=3, var>0.01, angles>=2): -3

## Test Priorities
1. computeBotScore() -- all 30+ signals
2. Challenge state transitions
3. Analytics SQL (human/bot counts, period filtering)
4. Device targeting (allowed_devices check)
5. Click deduplication logic
6. Post-challenge score recalculation
```

## 8.3 api-doc-agent.md

**File:** `.claude/agents/api-doc-agent.md`
**Tools:** Read, Write, Glob, Grep

```
---
name: api-doc-agent
description: API documentation generator for link-tracker. Reads route
  files, extracts endpoints, and generates markdown docs in the format
  consumed by campaign automation AI agents.
model: sonnet
tools: Read, Write, Glob, Grep
---

# Link Tracker -- API Documentation Agent

You generate and update API documentation by reading the actual
route files in server/routes/.

## Output Format

Generate markdown with:
- HTTP method and path
- Query parameters table (param, type, default, description)
- Request body JSON example
- Response JSON example with all fields
- Field descriptions for enums (challenge_status values, etc.)
- curl examples for key endpoints

## Route Files to Scan
- server/routes/apilinks.js -- Campaign API (primary)
- server/routes/links.js -- Link CRUD
- server/routes/analytics.js -- Analytics endpoints
- server/routes/webhooks.js -- Webhook management
- server/routes/auth.js -- Authentication
- server/routes/admin.js -- Admin endpoints
- server/routes/campaigns.js -- Campaign CRUD

## Key Conventions
- All endpoints return { success: true, ... } or { error: "msg" }
- Auth: Bearer token or X-API-Key header
- Pagination: { page, limit, total, pages }
- Bot score fields: bot_score (0-100), is_bot (0/1),
  challenge_status (null/passed/blocked/expired/device_blocked)
- Device targeting: allowed_devices field on campaigns
  (all/mobile/desktop/tablet/mobile,tablet/desktop,tablet)

## Output Location
Save to: API-Links-Documentation.md (project root)
```

## 8.4 deploy-agent.md

**File:** `.claude/agents/deploy-agent.md`
**Tools:** Read, Write, Edit, Glob, Grep, Bash

> **Note:** Focused on deploy + rollback ONLY. Monitoring is handled by monitoring-agent (Section 8.5).

```
---
name: deploy-agent
description: Deployment and rollback agent for link-tracker.
  Manages Windows Server + NSSM deploy pipeline, ZIP packaging,
  code verification, post-deploy health check, and automatic rollback.
model: sonnet
tools: Read, Write, Edit, Glob, Grep, Bash
---

# Link Tracker -- Deploy Agent

You manage the deployment pipeline and rollback for link-tracker
on Windows Server. Focused on deploy + rollback ONLY -- monitoring
is handled by monitoring-agent.

## Environment
- Production: Windows Server, NSSM service "Link Tracker"
- NSSM path: C:\Users\Administrator\Desktop\nssm\nssm-2.24\win64\nssm.exe
- App dir: C:\Users\Administrator\Desktop\link-tracker
- Deploy script: deploy-update.cmd
- ZIP tool: System.IO.Compression.ZipFile (NOT Compress-Archive)
- Dashboard build: node views/sectionsv3/assemble.js --save

## Deploy Pipeline (7 steps)
1. Create backup ZIP
2. Stop server (NSSM stop + triple taskkill + verify)
3. Clean stale flat files + extract ZIP (relative paths)
4. Verify code on disk (PowerShell -SimpleMatch checks)
5. Copy ipapi.db if newer
6. Start server (NSSM start + HTTP health check)
7. Run migrations if present

## Post-Deploy Health Check
- HTTP 200 on /health (NOT /login -- use the dedicated health endpoint)
- node.exe process running with fresh StartTime
- If health check fails -> trigger automatic rollback

## Automatic Rollback
- On health check failure, restore from backup ZIP created in step 1
- Stop server, extract backup ZIP, restart server
- Verify health again after rollback
- Report rollback outcome (success/failure) with details

## Known Issues
- Compress-Archive strips directory paths (use ZipFile)
- Expand-Archive dumps files flat (14 stale files incident)
- NSSM not on PATH (use full path)
- Nested if blocks crash CMD (use goto :label pattern)

## ZIP File List
server/utils/ip-threat.js, server/utils/helpers.js,
server/index.js, server/routes/redirect.js,
server/routes/apilinks.js, server/database.js,
views/v3/dashboard.html, views/sectionsv3/inline.js,
views/sectionsv3/layout.html, views/sectionsv3/modals.html,
views/sectionsv3/assemble.js, public/tabler/css/tabler-flags.min.css
```

## 8.5 monitoring-agent.md

**File:** `.claude/agents/monitoring-agent.md`
**Tools:** Read, Glob, Grep, Bash

```
---
name: monitoring-agent
description: Read-only monitoring and observability agent for link-tracker.
  Audits 16 background timers, health endpoint, memory/cache status,
  and logging gaps. Never modifies code or config.
model: sonnet
tools: Read, Glob, Grep, Bash
---

# Link Tracker -- Monitoring Agent

WARNING: This agent is READ-ONLY. Never modify code, config, or data.
Inspect, audit, and recommend only.

## Background Timers (16 total)

### index.js (13 timers)

| Timer | Interval | Line | Purpose |
|-------|----------|------|---------|
| WS rate limit reset | 10s | 628 | Reset WebSocket rate counters |
| WS ping keepalive | 30s | 641 | Ping dashboard clients |
| Anomaly detection | 10min | 700 | Click anomaly detection |
| Fraud learning | 5min | 730 | Update fraud patterns |
| Click rate cleanup | 30min | 764 | Prune click rate data |
| Memory watchdog | 60s | 800 | Prune caches if RSS > 1GB |
| Daily scheduler | 1h | 1045 | Purge logs, tokens |
| Honeypot flag cleanup | 1h | 1070 | Remove stale flags |
| Challenge timeout | 5min | 1086 | Expire pending challenges |
| Blueprint confidence decay | 7 days | 1110 | Decay bot blueprint scores |
| Retroactive bot check | 2min | 1141 | Re-evaluate recent clicks |
| SSL renewal | 24h | 1159 | Renew certificates |
| Daily cleanup | 24h | 1045 | Purge old data |

### performance.js (3 timers)

| Timer | Interval | Line | Purpose |
|-------|----------|------|---------|
| ClickBatcher flush | 2s | 53 | Batch-write clicks to SQLite |
| AnalyticsCache TTL cleanup | 60s | 158 | Evict stale analytics cache |
| LinkCache eviction | 30s | 284 | Remove expired link cache |

## Responsibilities

1. **Timer health audit** -- verify all 16 timers are registered and running
2. **Memory monitoring** -- check RSS, cache sizes, ClickBatcher queue depth
3. **Error pattern detection** -- scan console output for recurring errors
4. **/health endpoint audit** -- verify /health returns accurate status

## Key Files

- server/index.js -- 13 timers, /health endpoint
- server/performance.js -- 3 timers (ClickBatcher, caches)
- server/utils/helpers.js -- bot scoring (affects timer behavior)

## Rules

1. NEVER modify any file -- read-only inspection only
2. Report findings as structured recommendations
3. Flag any timer missing error handling (try/catch)
4. Flag any timer with no failure visibility
5. Recommend Pino structured logging where console.log is used
6. Bash tool is for READ-ONLY commands only: ps, tail, cat, grep, curl /health, du, free, netstat, wc. NEVER run commands that modify files, services, config, or data. This includes: no kill, no rm, no nssm start/stop/restart, no service control, no writes, no migrations, no database modifications. If an action requires modification, STOP and hand off to the appropriate agent (deploy-agent for service control, backend-architect for code fixes) with a clear recommendation.
```

## 8.6 frontend-agent.md

**File:** `.claude/agents/frontend-agent.md`
**Tools:** Read, Write, Edit, Glob, Grep, Bash

```
---
name: frontend-agent
description: Frontend specialist for link-tracker dashboard. Works with
  vanilla JS, Tabler (Bootstrap 5), and the assemble.js build pipeline.
  Handles inline.js, modals, section templates, and WebSocket live feed.
model: sonnet
tools: Read, Write, Edit, Glob, Grep, Bash
---

# Link Tracker -- Frontend Agent

You build and maintain the link-tracker dashboard UI.

## Architecture
- Framework: Vanilla JavaScript (NO React/Vue/Angular)
- UI Library: Tabler (Bootstrap 5 based) dark theme
- Charts: ApexCharts (renderChart helper)
- Maps: JSVectorMap (renderWorldMap, renderHeatmapChart)
- Build: views/sectionsv3/ source -> assemble.js -> views/v3/dashboard.html

## Key Files
- views/sectionsv3/inline.js (3,245 lines) -- ALL dashboard JS
- views/sectionsv3/modals.html -- 50+ modal dialogs
- views/sectionsv3/layout.html -- sidebar, main container
- views/sectionsv3/ui.html -- shared UI components
- views/sectionsv3/assemble.js -- build script

## Conventions
- Escape HTML: escHtml(text)
- API calls: API.get/post/put/delete (returns parsed JSON)
- Modals: uiForm(title, fields), uiPrompt(msg, default), uiConfirm(msg)
- Toast: showToast(message, type?)
- Tables: buildPagination(page, pages, loadFn)
- Sections: showSection(name) for navigation
- Numbers: formatNumber(n)

## After Every Change
Run: node views/sectionsv3/assemble.js --save
This compiles all section files into views/v3/dashboard.html

## WebSocket Live Feed
- Endpoint: /dashboard (WS upgrade)
- Auth: JWT from cookie
- Messages: { type: 'click', data: clickData }
- Used for real-time click notifications on dashboard

## Anti-patterns to Avoid
- Do NOT suggest React, Vue, or any framework migration
- Do NOT use jQuery (vanilla JS only)
- Do NOT create separate .js files (everything in inline.js)
- Do NOT forget to run assemble.js after changes
- Use _analyticsRequestId pattern for stale request prevention
- Use clearApiAnalyticsUI() when switching analytics views
```

---

# 9. Installation Plan

## 9.1 Prerequisites

- Claude Code CLI installed and authenticated
- Git installed (for cloning ready-made agent repos)
- Internet access (for fetching .md files from GitHub)

## 9.2 Step-by-Step Installation

### Step 1: Create agent directory

```
mkdir -p .claude/agents
```

### Step 2: Install ready-made agents

```
# security-auditor from iannuttall (NOTE: add model: sonnet to frontmatter)
curl -o .claude/agents/security-auditor.md \
  https://raw.githubusercontent.com/iannuttall/claude-agents/main/agents/security-auditor.md

# backend-architect from lst97
curl -o .claude/agents/backend-architect.md \
  https://raw.githubusercontent.com/lst97/claude-code-sub-agents/main/agents/development/backend-architect.md

# performance-engineer from lst97
curl -o .claude/agents/performance-engineer.md \
  https://raw.githubusercontent.com/lst97/claude-code-sub-agents/main/agents/infrastructure/performance-engineer.md
```

### Step 3: Create custom agents

Copy each draft from Section 8 into its corresponding file:

- `.claude/agents/tech-lead.md` (Section 8.1)
- `.claude/agents/test-agent.md` (Section 8.2)
- `.claude/agents/api-doc-agent.md` (Section 8.3)
- `.claude/agents/deploy-agent.md` (Section 8.4)
- `.claude/agents/monitoring-agent.md` (Section 8.5)
- `.claude/agents/frontend-agent.md` (Section 8.6)

### Step 4: Verify installation

```
ls -la .claude/agents/
# Should show 9 .md files
```

### Step 5: Test each agent

```
# Test orchestrator
@tech-lead What agents are available?

# Test a specialist
@test-agent What would you test first in this project?

# Test a ready-made
@security-auditor Scan server/middleware/auth.js
```

## 9.3 Rollback Plan

Agents are just .md files — to rollback, simply delete the file:

```
rm .claude/agents/agent-name.md
```

No code changes, no config modifications, no side effects. Completely reversible.

---

# 10. Usage Guide

## 10.1 Invocation Methods

| Method | Syntax | When to Use |
|--------|--------|-------------|
| Direct | @agent-name task | When you know which specialist to call |
| Via orchestrator | @tech-lead task | When task spans multiple domains |
| Auto-detected | Just describe the task | Agent trigger descriptions match keywords |

## 10.2 Example Workflows

### Workflow A: Add a new feature (e.g., device targeting)

Tell the orchestrator:

```
@tech-lead Add allowed_devices field to campaigns with alert page for wrong device
```

The orchestrator will:

- Plan backend changes (database migration, API endpoints, redirect logic)
- Delegate UI work to `@frontend-agent`
- Delegate tests to `@test-agent`
- Trigger `@api-doc-agent` to update documentation
- Trigger `@deploy-agent` to update deploy verification

### Workflow B: Fix a bug (e.g., wrong click counts)

```
@tech-lead Human click count is wrong in campaign analytics
```

The orchestrator will:

- Read the analytics SQL in apilinks.js
- Identify the bug, coordinate the fix
- Delegate regression test to `@test-agent`
- Trigger deploy pipeline

### Workflow C: Security audit

```
@security-auditor Full OWASP scan of server/routes/ and middleware/
```

Direct invocation — no orchestrator needed for single-domain tasks.

## 10.3 Maintenance

- **Weekly:** Review agent performance — are they producing useful output?
- **Monthly:** Check ready-made agent repos for updates
- **On roster change:** Update tech-lead.md subagent roster table
- **On feature addition:** Run `@api-doc-agent` to update docs

---

# 11. Self-Review Summary

## 11.1 Issues Found and Fixed

| # | Issue | Severity | Resolution |
|---|-------|----------|------------|
| 1 | Missing Need: Monitoring (16 background timers invisible) | **Critical** | Added Need #8, dedicated monitoring-agent |
| 2 | Table count: reported 32, actual 33 | **High** | Corrected to 33 |
| 3 | Route files: reported 14, actual 15 | **High** | Corrected to 15 |
| 4 | lst97 code-reviewer uses Haiku model | **High** | Replaced with backend-architect (Sonnet) |
| 5 | Missed lst97 backend-architect agent | **High** | Added as Need #5 recommendation |
| 6 | WebSocket live feed not addressed | **High** | Added to frontend-agent scope |
| 7 | Sync webhook delivery not flagged | **High** | Noted in performance scope |
| 8 | 0xfurai agents don't specify tools | **High** | Corrected description |

## 11.2 Confidence Scores

| Agent | Score | Justification |
|-------|-------|---------------|
| test-agent (Custom) | **9/10** | Zero tests + unique domain logic = clear custom need. Bot scoring first priority |
| tech-lead (Custom) | **9/10** | Only 1 existing agent delegates; it solves wrong problem. Ambiguity fallback added |
| deploy-agent (Custom) | **9/10** | Windows+NSSM stack is unique. ZIP/rollback focused after monitoring split |
| api-doc-agent (Custom) | **9/10** | AI agents need specific format. No existing agent generates it |
| monitoring-agent (Custom) | **8/10** | 16 timers with zero failure detection. Read-only scope keeps it simple |
| frontend-agent (Custom) | **8/10** | Vanilla JS + Tabler is unusual. Risk: scope creep with WS |
| security-auditor (Ready) | **8/10** | Verified exists with write tools. Note: missing model field -- add model: sonnet |
| backend-architect (Ready) | **7/10** | Good fit but may overlap with orchestrator planning. Now includes migration guidance |
| performance-engineer (Ready) | **7/10** | Sonnet model adequate. May miss SQLite-specific tuning |

## 11.3 Known Limitations

- Agent prompts are static — they don't auto-update when the codebase changes
- Ready-made agents may receive upstream updates that break compatibility
- No automated way to verify agent output quality over time
- Orchestrator routing rules are keyword-based, not semantic — ambiguity fallback (ask user) mitigates misrouting

---

# 12. Decisions Applied

| # | Decision | Answer | Impact |
|---|----------|--------|--------|
| 1 | Testing focus: bot scoring or analytics SQL first? | Bot scoring first | test-agent prioritizes computeBotScore() and challenge flow |
| 2 | Deploy agent rollback? | Yes, automatic rollback from backup ZIP | deploy-agent restores backup ZIP on health check failure |
| 3 | API doc format: markdown or OpenAPI? | Markdown | api-doc-agent generates markdown for AI agent consumption |
| 4 | Agent installation scope: global or project? | Project-level (.claude/agents/) | Agents scoped to link-tracker only |
| 5 | Frontend agent + WebSocket? | Keep in frontend-agent | WebSocket live feed stays in frontend-agent scope |
| 6 | Code-reviewer alongside backend-architect? | No, backend-architect only | Haiku model too shallow; Sonnet backend-architect sufficient |
| 7 | Monitoring: fold into deploy or separate? | Separate monitoring-agent | Read-only agent, zero coupling with deploy |
| 8 | CLAUDE.md creation? | Yes | Project conventions file referenced by all agents |
| 9 | Security-auditor model? | Add model: sonnet | Explicit model prevents inheriting wrong default |

---

# 13. Appendix A: All GitHub Agents Considered

| Agent | Repo | Verdict | Reason |
|-------|------|---------|--------|
| security-auditor | iannuttall | **Accepted** | Write tools, comprehensive OWASP coverage |
| backend-architect | lst97 | **Accepted** | Sonnet model, API/schema design focus |
| performance-engineer | lst97 | **Accepted** | Full-stack perf, Sonnet model |
| code-reviewer | lst97 | **Rejected** | Uses Haiku model -- too shallow for meaningful review |
| agent-organizer | lst97 | **Rejected** | Plan-only, no Agent tool, uses Haiku |
| sqlite-expert | 0xfurai | **Optional** | Useful but merged into test-agent scope |
| express-expert | 0xfurai | **Rejected** | Generic Express knowledge, no unique value-add |
| owasp-top10-expert | 0xfurai | **Rejected** | Reference material, iannuttall's is more actionable |
| team-lead | wshobson | **Rejected** | Coding team manager, not a dispatcher. Requires TeamCreate tool |
| tech-lead-orchestrator | vijaythecoder | **Rejected** | No Agent tool, indirect delegation only |
| multi-agent-coordinator | VoltAgent | **Rejected** | Theoretical, no Agent tool, no practical delegation |
| workflow-orchestrator | VoltAgent | **Rejected** | State machine design, not operational |
| security-auditor | VoltAgent | **Rejected** | Read-only (no fix capability), compliance-heavy |
| deployment-engineer | VoltAgent | **Rejected** | Linux/Docker focused, incompatible with Windows/NSSM |
| test-automator | lst97 | **Rejected** | Generic, custom test-agent has domain knowledge |
| frontend-developer | lst97 | **Rejected** | React/Vue focused, incompatible with vanilla JS |
| project-task-planner | iannuttall | **Rejected** | Planning only, no delegation capability |

---

# 14. Appendix B: Repository References

| Repository | URL | Last Verified |
|-----------|-----|---------------|
| wshobson/agents | `github.com/wshobson/agents` | April 2026 |
| VoltAgent/awesome-claude-code-subagents | `github.com/VoltAgent/awesome-claude-code-subagents` | April 2026 |
| 0xfurai/claude-code-subagents | `github.com/0xfurai/claude-code-subagents` | April 2026 |
| iannuttall/claude-agents | `github.com/iannuttall/claude-agents` | April 2026 |
| lst97/claude-code-sub-agents | `github.com/lst97/claude-code-sub-agents` | April 2026 |
| rahulvrane/awesome-claude-agents | `github.com/rahulvrane/awesome-claude-agents` | April 2026 |
| vijaythecoder/awesome-claude-agents | `github.com/vijaythecoder/awesome-claude-agents` | April 2026 |

---

# 15. Appendix C: Glossary

| Term | Definition |
|------|-----------|
| Subagent | A specialized AI agent spawned by Claude Code's Agent tool to handle a specific subtask. Runs in its own context and returns results to the parent. |
| Orchestrator | A meta-agent whose job is to receive requests, decompose them into subtasks, delegate to specialist subagents, and aggregate results. Does not implement directly. |
| YAML Frontmatter | The metadata block at the top of an agent .md file, enclosed in --- delimiters. Contains name, description, model, and tools fields. |
| System Prompt | The instructions given to an AI agent that define its role, knowledge, constraints, and behavior. The content of the .md file after the frontmatter. |
| Context Window | The maximum amount of text an AI model can process in a single conversation. Subagents help by isolating work into separate, smaller contexts. |
| Tool Permissions | The set of tools (Read, Write, Edit, Bash, Agent, etc.) available to a subagent. Defined in the frontmatter. Follows least-privilege principle. |
| Agent Tool | The Claude Code tool that spawns a new subagent. Only agents with this tool in their permissions can delegate to other agents. |
| Ready-made Agent | A pre-written agent .md file from a GitHub repository. Can be installed by copying the file. May need model or tool overrides. |
| Custom Agent | An agent .md file written specifically for your project, encoding domain-specific knowledge, conventions, and file paths. |

---

*Confidential -- Prepared for Link Tracker Project*
