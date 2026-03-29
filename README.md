# Qimah Eyes

Analytics data pipeline that powers the Qimah Eyes dashboard. Ingests data from WordPress webhooks, external APIs (Moyasar, Bunny, Resend, ClickUp, PostHog), transforms it through n8n workflows, computes derived metrics via nightly Node.js scripts, and stores everything in Postgres 16.

## Architecture

```
DATA SOURCES                          LAYER 1: INGESTION (n8n)
┌─────────────────────────┐      ┌──────────────────────────────┐
│ WP Webhooks (HMAC)      │─────>│ Webhook Workflows            │
│ WP REST API             │      │ - HMAC validate              │
│ Moyasar (payments)      │─────>│ - Log raw event              │
│ Bunny (video CDN)       │      │ - Transform + upsert         │
│ Resend (email)          │      │                              │
│ ClickUp (recruitment)   │      │ API Pull Workflows           │
│ PostHog (product)       │      │ - Cron-triggered, paginated  │
│ qimah-email log         │      │ - Transform + write          │
└─────────────────────────┘      └──────────────┬───────────────┘
                                                │
                                 LAYER 2: COMPUTATION (Node.js)
                                 ┌──────────────┴───────────────┐
                                 │ gate.js     - wait for L1    │
                                 │ compute.js  - derived metrics│
                                 │ insights.js - rule engine    │
                                 │ ai-weekly.js - Claude summary│
                                 │ cleanup.js  - retention      │
                                 └──────────────┬───────────────┘
                                                │
                                 LAYER 3: STORAGE (Postgres 16)
                                 ┌──────────────┴───────────────┐
                                 │ 33 domain tables             │
                                 │ events (raw log)             │
                                 │ pulse (real-time KPIs)       │
                                 │ cron_health (job tracking)   │
                                 │ dead_letters (error queue)   │
                                 └──────────────────────────────┘
```

**Key principle:** n8n moves data in, scripts compute derived data, Postgres stores everything. No logic spans both layers.

## Data Domains

| Domain | Sources | Webhook Events | Nightly Computations |
|--------|---------|---------------|---------------------|
| Revenue | WooCommerce, Moyasar | `commerce.order.*`, `commerce.refund.*` | Revenue aggregations, AOV, refund rates |
| Enrollment | LearnDash, WooCommerce | `enrollment.*`, `batch.*` | Cohort stats, conversion funnels |
| Activity | qimah-sec, qimah-telemetry | `activity.*`, `session.*` | Session patterns, engagement scores |
| Gamification | qimah-gamification | `points.*`, `streak.*` | Streak history, leaderboard deltas |
| Video | Bunny CDN, QVM | `video.*` | Watch-time aggregations, completion rates |
| Communication | Resend, qimah-email | `email.*` | Delivery rates, open/click metrics |
| Recruitment | ClickUp | API pull (nightly) | Pipeline stage durations |
| Product | PostHog | API pull (nightly) | Feature usage, funnel analysis |

## Webhook Contract

Single endpoint: `POST /webhook/qimah`

- **Auth**: HMAC-SHA256 signature in `X-Qimah-Signature` header
- **Routing**: `event_type` field determines processing branch
- **Idempotency**: `ON CONFLICT DO UPDATE` upserts on natural keys
- **Error handling**: Failed webhooks go to `dead_letters` table with retry for critical paths (revenue, security)

## Nightly Pipeline

Triggered by n8n cron, orchestrated by `gate.js`:

1. **API pulls** run in parallel (Moyasar, Bunny, ClickUp, PostHog, WP REST)
2. **Gate** waits for all pulls to report completion in `cron_health`
3. **Compute** runs derived metrics (streak history, instructor scores, cohort stats, session patterns, pulse deltas)
4. **Insights** evaluates 20 rule-engine conditions, generates alerts
5. **AI Weekly** (Sundays) sends summary data to Claude Sonnet for narrative analysis
6. **Cleanup** enforces event retention policy, runs `VACUUM ANALYZE`

## Tech Stack

| Layer | Stack |
|-------|-------|
| Ingestion | n8n (self-hosted), 14 domain-grouped workflows |
| Computation | Node.js scripts, triggered by n8n Execute Command |
| Storage | Postgres 16 (33 tables + system tables) |
| AI | Claude Sonnet (weekly narrative summaries) |
| Infra | Hetzner VPS, systemd, Tailscale mesh |

## Design Docs

- **Data pipeline spec**: `docs/superpowers/specs/2026-03-15-qimah-eyes-data-pipeline-design.md` (in plugins repo)
- **Implementation plan**: `docs/superpowers/plans/2026-03-15-qimah-eyes-data-pipeline.md` (in plugins repo)
- **Dashboard UI spec** (parent): `docs/superpowers/specs/2026-03-13-god-dashboard-design.md` (in plugins repo)

## Status

**Pre-development** - spec and implementation plan complete, not yet built.
