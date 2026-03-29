# Qimah Eyes - Data Pipeline Design Spec

**Date:** 2026-03-15
**Status:** Draft
**Author:** Claude (brainstormed with M)
**Parent spec:** `2026-03-13-god-dashboard-design.md` (dashboard UI + schema + insight engine)

---

## 1. Summary

The ingestion and ETL pipeline that feeds Qimah Eyes (the analytics dashboard). Defines how data flows from WordPress, external APIs, and PostHog into Postgres - webhook reception, API pulling, transformation, nightly computation, error handling, and backfill.

This spec covers the data engineering layer only. For the dashboard UI, Postgres schema (33 tables), insight engine rules, and frontend design, see the parent dashboard spec.

---

## 2. Design Decisions

Decisions made during brainstorming, with rationale:

| Decision | Choice | Why |
|----------|--------|-----|
| Orchestration tool | n8n | Team is comfortable, students learning it, visual and teachable |
| Error handling | Hybrid: DLQ-retried paths (revenue, security) get dead letter queue + retry; everything else best-effort + nightly backfill | Revenue/security data loss is unacceptable; learning activity gaps can wait for nightly catch-up |
| Workflow organization | Domain-grouped (14 workflows, not 23+) | n8n's flat workflow list is hard to navigate at scale; domain grouping maps to dashboard sections |
| Nightly job ordering | Hybrid: time-based API pulls (parallel), gate script checks completion before triggering computations | API pulls are independent (parallel is safe); computations genuinely depend on fresh data |
| Idempotency | Database-level via `ON CONFLICT DO UPDATE` (upserts) | Natural keys exist on most tables; no extra queries, no WP-side changes needed |
| Transformation location | All in n8n for V1 | Transforms are simple (TZ conversion, field mapping, one HTTP lookup); separate helper API not justified for ~10 transforms |
| Nightly computations | Standalone Node.js scripts triggered by n8n via Execute Command | SQL-heavy logic (window functions, aggregations) is painful in n8n's Code node editor; scripts are testable and version-controlled |
| Initial data | Full one-time backfill script | Dashboard is useless with empty tables; historical revenue and enrollment data needed from day one |
| Project name | Qimah Eyes | Renamed from "GOD Dashboard" |

---

## 3. Architecture Overview

Three layers with clear separation of concerns:

```
+---------------------------------------------------------+
|                     DATA SOURCES                         |
|  WP Webhooks | WP REST API | Moyasar | Bunny | Resend   |
|  ClickUp | PostHog | qimah-email log                    |
+---------+------------------+----------------------------+
          |                  |
          v                  v
+---------------------------------------------------------+
|             LAYER 1: INGESTION (n8n)                     |
|                                                          |
|  Webhook Workflows      |  API Pull Workflows            |
|  - Receive POST         |  - Cron-triggered              |
|  - Validate HMAC        |  - Paginate API responses      |
|  - Log to events table  |  - Transform + write           |
|  - Validate payload     |  - Report to cron_health       |
|  - Transform fields     |                                |
|  - Upsert domain table  |                                |
|  - Update pulse         |                                |
|  - Report cron_health   |                                |
+---------+----------------------------------------------+
          |
          v
+---------------------------------------------------------+
|         LAYER 2: COMPUTATION (Node.js scripts)           |
|                                                          |
|  gate.js          |  Waits for all L1 pulls to           |
|                   |  report done in cron_health           |
|                   |                                       |
|  compute.js       |  streak_history, instructor_scores,   |
|                   |  cohort_stats, session_patterns,     |
|                   |  summary, pulse deltas                |
|                   |                                       |
|  insights.js      |  Rule engine (20 rules)               |
|  ai-weekly.js     |  Weekly Claude Sonnet analysis         |
|  cleanup.js       |  Event retention, VACUUM ANALYZE       |
+---------+----------------------------------------------+
          |
          v
+---------------------------------------------------------+
|         LAYER 3: STORAGE (Postgres 16)                   |
|                                                          |
|  33 domain tables + dead_letters + work_tasks             |
|  events (raw log) | pulse | cron_health                  |
+---------------------------------------------------------+
```

**Key principle:** n8n moves data in, scripts compute derived data, Postgres stores everything. No logic spans both - if n8n writes a row, scripts read it; they never both write to the same table in the same pipeline run.

**Exception:** `pulse` table - webhooks update pulse in real-time (n8n), and nightly scripts also refresh pulse deltas. Safe because they write different keys (`revenue_today` vs `revenue_7d_delta`).

---

## 4. Webhook Ingestion (Layer 1a)

Single webhook endpoint on n8n. Routes by `event_type` field in payload.

### 4.1 Standard Pipeline

Every webhook follows this flow:

```
POST /webhook/qimah
        |
        v
+- HMAC Validate --+
| X-Qimah-Signature|----- FAIL ----> Log to dead_letters, return 401
+--------+---------+
         v
+- Log Raw Event --+
| INSERT events    |  (always, before any transform)
+--------+---------+
         v
+- Validate Shape -+
| Required fields? |----- FAIL ----> Log to dead_letters, return 200
| Types correct?   |                 (200 so WP doesn't retry garbage)
+--------+---------+
         v
+- Switch on event_type -----------------------------------+
| commerce.*  -> Revenue branch                            |
| activity.*, session.*, points.*, streak.* -> Activity    |
| user.*      -> User branch                               |
| feedback.*  -> Learning branch                           |
| security.*  -> Security branch                           |
| circle.*    -> Community branch                          |
| booking.*   -> Booking branch (generic log only)         |
| unknown     -> dead_letters                              |
+--------+-------------------------------------------------+
         v
+- Transform ------+
| TZ -> UTC        |
| Field mapping    |
| Lookups (if any) |
+--------+---------+
         v
+- Upsert Domain --+
| ON CONFLICT      |----- ERROR ----> dead_letters
| DO UPDATE        |
+--------+---------+
         v
+- Update Pulse ---+
| Upsert pulse row |
+--------+---------+
         v
+- Report Health --+
| Upsert cron_    |
| health row      |
+------------------+
```

### 4.2 Webhook Workflow Inventory

6 domain-grouped n8n workflows (consolidates the 10 individual webhook workflows in the parent spec into 6 by domain - this spec supersedes the parent's webhook workflow count):

| Workflow | Events handled | DLQ + retry? | Write targets |
|----------|---------------|-------------|---------------|
| Revenue | `commerce.order_paid`, `commerce.order_refunded` | Yes | orders, order_items, customers, coupons (aggregate stats upsert), pulse |
| Activity & Sessions | `activity.logged`, `topic.completed`, `session.started`, `points.awarded`, `activity.streak_milestone` | No (except `session.started` - see note) | daily_activity, pulse |
| Users | `user.registered` | No | user_profiles, conversions (+ PostHog `identify` call to bridge anonymous_id), pulse |
| Security | `device.*`, `session.*`, `sharing.*` | Yes | sharing_alerts, pulse (raw events logged to `events` table; `session_patterns` computed nightly) |
| Learning | `feedback.submitted` | No | feedback (vote, instructor_id, comment), pulse |
| Community & Bookings | `circle.*`, `booking.*` | No | circles, circle_activity, pulse |

**Note on `session.started`:** The parent dashboard spec marks `session.started` as Phase 1 critical (needed for Pulse). The Activity workflow is non-critical overall, but `session.started` events should be treated as important for data freshness. Nightly API pull backfills any gaps, so no DLQ needed - but monitor via cron_health if drop-off is detected.

### 4.3 Error Handling by Tier

**DLQ-retried paths (Revenue + Security):**
- On transform/write failure: write full payload to `dead_letters` with error message
- n8n retry: 3 attempts with exponential backoff (1min, 5min, 15min)
- If all retries fail: stays in `dead_letters`, nightly insight rule alerts ("N unresolved dead letters")

**Best-effort paths (Activity, Users, Learning, Community):**
- Single attempt, failures go to `dead_letters`
- Most tables have a nightly API pull that backfills gaps
- No retry (data arrives again via nightly pull)

**Accepted risk:** `daily_activity` has no API pull backfill path - it is populated exclusively by webhooks (`activity.logged`, `topic.completed`, `session.started`). If webhooks fail for a day, that day's activity data is lost. This is acceptable because: (1) webhook delivery is reliable in practice, (2) the impact is limited to streak/retention accuracy for one day, and (3) adding a WP-side daily activity API would require significant WP plugin changes not justified at this scale.

**Terminology note:** "DLQ-retried" vs "best-effort" describes webhook error handling tiers. Separately, the gate script (Section 6.1) classifies API pulls as "required" vs "optional" for go/no-go decisions. Revenue webhooks are DLQ-retried, and WP Learning pulls are gate-required - related but distinct concepts.

### 4.4 Dead Letters Table

```sql
CREATE TABLE dead_letters (
  id            SERIAL PRIMARY KEY,
  event_type    VARCHAR(100),
  payload       JSONB NOT NULL,
  error_message TEXT,
  source        VARCHAR(30),      -- wp_webhook, api_pull
  retries       INT DEFAULT 0,
  resolved      BOOLEAN DEFAULT false,
  created_at    TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_dead_letters_unresolved ON dead_letters(resolved, created_at) WHERE NOT resolved;
```

**Note:** This is table #34, in addition to the 33 domain tables in the parent dashboard spec.

### 4.5 Payload Validation

Each webhook workflow validates required fields before processing. Validation is lightweight - check presence and type of critical fields, not full schema enforcement. Example for `commerce.order_paid`:

- Required: `order_id` (int), `total` (number > 0), `user_id` (int), `items` (array)
- If missing: write to `dead_letters` with `error_message = "missing required field: order_id"`, return 200

### 4.6 updated_at on Upsert Targets

Tables that receive `ON CONFLICT DO UPDATE` should include `updated_at TIMESTAMP DEFAULT NOW()` and set it in the update clause. This tracks when a row was last refreshed vs. when it was created.

Applies to: `orders`, `customers`, `user_profiles`, `daily_activity`, `course_snapshots`, `topic_stats`, `coupons`, `cohorts`, `circles`, `conversions`, `payment_failures`, `email_campaigns`, `video_pipeline`, `visitors`, `pulse`.

**Note on `work_tasks`:** This table is not defined in the parent dashboard spec schema. It is a new table introduced by the pipeline to store ClickUp task snapshots (id, status, assignee, list, dates). DDL to be added to `sql/schema.sql` during implementation.

**Note on `cron_health.status`:** Parent schema defines `ok`, `warning`, `failed`. V1 of the pipeline uses only `ok` and `failed`. The `warning` status is reserved for future use (e.g., "completed but slow" or "completed with partial data").

---

## 5. API Pull Ingestion (Layer 1b)

Seven n8n workflows, cron-triggered nightly. All independent - run in parallel.

### 5.1 API Pull Workflow Inventory

| Workflow | Source | Auth | Pagination | Writes to |
|----------|--------|------|------------|-----------|
| WP Learning | `GET /analytics/enrollments` + `GET /analytics/feedback` + `GET /wp/v2/sfwd-courses?_fields=id,author` | `X-Qimah-API-Key` | Cursor-based | course_snapshots (enrollment counts + avg_feedback_pct), topic_stats |
| WP Users | `GET /leaderboard` | `X-Qimah-API-Key` | Page-based | user_profiles |
| Moyasar | `GET /v1/payments?status=failed` | Bearer token | Page-based | payment_failures |
| Bunny | Bunny Stream API + `GET /video-manager/stats` | API key header | Per-library | video_pipeline, topic_stats |
| Resend | Resend API (emails, domains) | Bearer token | Cursor-based | email_campaigns, email_events |
| ClickUp | ClickUp API (lists, tasks) | API token | Page-based | work_tasks, applicants (instructor pipeline), pipeline_snapshots |
| PostHog | PostHog API (events, persons, insights) | Personal API key | Next URL | visitors, funnels, conversions, video_events (see 5.6) |

### 5.2 Standard Pattern

Each API pull workflow:
1. Fetch paginated data (loop until exhausted)
2. Transform in Code node (field mapping, TZ normalization)
3. Upsert to domain table(s) via `ON CONFLICT`
4. Report status to `cron_health` (`ok` or `failed`)

### 5.3 Scheduling

- **00:00 UTC:** 6 API pull workflows start simultaneously (WP Learning, WP Users, Moyasar, Bunny, Resend, ClickUp)
- **00:30 UTC:** PostHog pull starts (delayed because PostHog's own nightly aggregation needs time to complete first)
- **00:50 UTC:** Gate script starts polling cron_health (see Section 6.1)

### 5.4 Error Handling

If an API pull fails (timeout, auth error, rate limit):
- n8n retries 3x with backoff
- On final failure: writes to `dead_letters` with `source='api_pull'`, reports `failed` to `cron_health`
- Gate script (Section 6.1) sees the `failed` status and decides whether to proceed with stale data or skip

### 5.5 Grouping Rationale

WP Learning combines enrollments + feedback + course authors into one workflow because they all write to the same tables and share the same API key. The rest are one workflow per external service.

### 5.6 PostHog -> video_events Transform

PostHog stores Bunny iframe engagement as `$autocapture` events on the embedded video player. The PostHog API pull extracts these and transforms them:

- **Source:** PostHog events API, filter by `$autocapture` on Bunny iframe elements + any custom `video_*` events
- **Transform:** Group by user_id + topic_id + date. Aggregate: total watch_duration_sec (sum of session lengths), pct_watched (max position reached / video duration), play_count (distinct sessions)
- **Write:** Upsert to `video_events` table keyed on `(user_id, topic_id, date)`. **Schema change required:** parent spec's `video_events` has `created_at TIMESTAMP` but no `date` column or UNIQUE constraint. Add `date DATE` column and `UNIQUE(user_id, topic_id, date)` constraint during implementation
- **Fallback:** If PostHog autocapture proves insufficient for iframe events, V2 adds explicit CEP-side tracking via `player.on('timeupdate')` -> PostHog custom event (per parent spec)

### 5.7 Tables with No API Pull (Webhook-Only or Admin-Seeded)

Some tables in the parent schema are populated outside the API pull workflows:

| Table | Write path | Notes |
|-------|-----------|-------|
| `feedback` | Webhook only (`feedback.submitted`) | No bulk feedback API; nightly WP Learning pull updates `course_snapshots.feedback_pct` from aggregate stats, not individual feedback rows |
| `circles`, `circle_activity` | Webhook only (`circle.*`) | No bulk circle API |
| `cohorts` | Admin-seeded (manual inserts) | Batch/campaign cohort definitions created by admin |
| `cohort_members` | Nightly computation or admin-seeded | Membership derived from user attributes (batch, campaign UTM) during nightly compute |
| `sharing_alerts` | Webhook only (`security.*`) | Forward-looking security data |
| `session_patterns` | Nightly computation (compute.js step 5) | Computed from raw `events` table, not written directly by webhooks |
| `discord_activity` | V2 (deferred) | Requires Discord bot deployment - see parent spec |

---

## 6. Nightly Computation Pipeline (Layer 2)

Triggered by n8n via `Execute Command` node. All scripts live in `scripts/` directory in the Qimah Eyes repo.

### 6.1 Orchestration Flow

```
00:00 UTC --- n8n triggers 6 API pull workflows (parallel)
             (WP Learning, WP Users, Moyasar, Bunny, Resend, ClickUp)
                     |
00:30 UTC --- n8n triggers PostHog pull (delayed for PostHog's own nightly aggregation)
                     |
                     v
              Each workflow reports to cron_health on completion
                     |
                     v
00:50 UTC --- n8n triggers: node scripts/gate.js
                     |
              gate.js polls cron_health every 60s:
              "Are all 7 API pulls status='ok' for today?"
                     |
              +-- YES --> exit 0
              +-- TIMEOUT (30min) --> exit 1 (n8n alerts)
              +-- PARTIAL FAILURE --> check which failed:
                    +-- Required --> exit 1
                    +-- Optional --> exit 0 (proceed with stale data)

              Gate classification:
              REQUIRED (failure blocks compute):
                WP Learning  - feeds instructor_scores, cohort_stats
                Moyasar      - feeds payment_failures (revenue accuracy)
              OPTIONAL (stale data acceptable):
                WP Users     - user_profiles refresh is cosmetic
                Bunny        - video_pipeline stats delay is tolerable
                Resend       - email stats delay is tolerable
                ClickUp      - work tasks + instructor pipeline delay is tolerable
                PostHog      - visitors/funnels can use yesterday's data
                     |
                     v
~01:00 UTC --- n8n triggers: node scripts/compute.js
              (approximate, depends on gate completion)
                     |
              Runs computations IN ORDER (dependencies):
              1. streak_history        (needs: daily_activity)
              2. user_profiles_streaks (needs: streak_history)
              3. instructor_scores (needs: course_snapshots, feedback, topic_stats)
              4. cohort_stats      (needs: cohort_members, daily_activity)
              5. session_patterns (needs: events)
              6. summary           (needs: all above)
              7. pulse deltas      (needs: all above)
                     |
              Each step: run SQL, report to cron_health
                     |
                     v
~01:30 UTC --- n8n triggers: node scripts/insights.js
                     |
              Runs 20 rules against fresh computed data
              Writes to insights table
                     |
                     v
Sunday 02:00 --- n8n triggers: node scripts/ai-weekly.js
                     |
              Collects cross-section data snapshot
              Sends to Claude Sonnet API
              Writes AI insights
                     |
                     v
03:00 UTC --- n8n triggers: node scripts/cleanup.js
                     |
              DELETE FROM events WHERE created_at < NOW() - INTERVAL '90 days'
              DELETE FROM dead_letters WHERE resolved AND created_at < NOW() - INTERVAL '30 days'
              VACUUM ANALYZE events
              VACUUM ANALYZE dead_letters
```

### 6.2 Scripts Directory Structure

```
scripts/
+-- gate.js              # Polls cron_health, decides go/no-go
+-- compute.js           # Runs all computations in dependency order
+-- insights.js          # 20 rule functions, writes to insights table
+-- ai-weekly.js         # Claude Sonnet analysis
+-- cleanup.js           # Retention enforcement
+-- backfill.js          # One-time historical data seeder
+-- lib/
|   +-- db.js            # Postgres connection pool (pg)
|   +-- health.js        # Shared cron_health reporter
+-- rules/
    +-- revenue.js       # Revenue drop, refund spike, etc.
    +-- learning.js      # Completion drop, feedback shift, etc.
    +-- retention.js     # Churn risk, streak decay, etc.
    +-- security.js      # Sharing alerts, device anomalies, etc.
    +-- operations.js    # Pipeline health, task bottlenecks, etc.
    +-- comms.js         # Email deliverability, bounce rate, etc.
    +-- community.js     # Circle activity drop, etc.
    +-- recruitment.js   # Pipeline stalls, etc.
    +-- marketing.js     # Funnel drop-off, conversion changes, etc.
```

### 6.3 compute.js Internals

Each computation is a single SQL query executed via `pg` client - not application logic. The script runs them sequentially in dependency order, reporting each step's status and duration to `cron_health`.

Example pattern:
```js
const steps = [
  { name: 'streak_history', fn: computeStreaks },
  { name: 'user_profiles_streaks', fn: updateUserStreaks },
  { name: 'instructor_scores', fn: computeInstructorScores },
  { name: 'cohort_stats', fn: computeCohortStats },
  { name: 'session_patterns', fn: computeSessionPatterns },
  { name: 'summary', fn: computeSummary },
  { name: 'pulse_deltas', fn: refreshPulseDeltas },
];

for (const step of steps) {
  const start = Date.now();
  try {
    await step.fn(db);
    await reportHealth(db, step.name, 'ok', Date.now() - start);
  } catch (err) {
    await reportHealth(db, step.name, 'failed', Date.now() - start, err.message);
    throw err; // Stop pipeline on failure
  }
}
```

Each `fn` contains a single parameterized SQL query (INSERT...ON CONFLICT, UPDATE, or INSERT...SELECT) with the computation logic in SQL, not JS.

---

## 7. Backfill Strategy

One-time script (`scripts/backfill.js`) run manually before go-live. Seeds historical data so the dashboard is useful from day one.

### 7.1 Tables to Backfill

| Table | Source | Depth | Notes |
|-------|--------|-------|-------|
| orders + order_items | WooCommerce REST API (`GET /wc/v3/orders`) | All time | Paginate through all completed/refunded orders |
| customers | Derived from orders | All time | Upsert per unique customer in order history |
| coupons | WooCommerce REST API (`GET /wc/v3/coupons`) | All time | Small dataset, full pull |
| user_profiles | WP REST API (`GET /wp/v2/users`) + leaderboard | All time | ~2K users |
| course_snapshots | `GET /analytics/enrollments` | Current state | Single snapshot, no history (API returns current counts) |
| topic_stats | `GET /analytics/enrollments` | Current state | Same pull as course_snapshots |
| conversions | WP user registration dates + PostHog | Best effort | Match by email/user_id where possible |
| email_campaigns + email_events | qimah-email delivery log (WP REST or direct DB) + Resend API | 90 days | WP plugin log has sent/delivered/bounced history |
| work_tasks + applicants + pipeline_snapshots (ClickUp) | ClickUp API (lists, tasks, statuses) | Active only | Team tasks + instructor pipeline state |
| video_pipeline | Bunny API | Current state | Active videos only |

### 7.2 Tables NOT Backfilled

Start from zero, accumulate naturally:
- `daily_activity` - no historical source (WP doesn't store daily activity aggregates)
- `streak_history` - depends on daily_activity
- `feedback` - no bulk API (webhook-only going forward)
- `sharing_alerts`, `session_patterns` - security events are forward-looking
- `circle_activity` - webhook-only
- `events` - raw log, no point backfilling

### 7.3 Script Design

- **Idempotent:** uses same `ON CONFLICT` upserts as production pipelines
- **CLI interface:** `node scripts/backfill.js --target=orders --dry-run` then `--confirm`
- **Per-table flags:** backfill incrementally, one table at a time
- **Rate-limited:** WooCommerce 1 req/sec, Resend 2 req/sec, ClickUp 100 req/min
- **Progress logging:** stdout with row counts per batch

---

## 8. Version Control & Deployment

### 8.1 Repository Structure

```
qimah-eyes/
+-- n8n/
|   +-- workflows/
|   |   +-- webhook-revenue.json
|   |   +-- webhook-activity.json
|   |   +-- webhook-security.json
|   |   +-- webhook-users.json
|   |   +-- webhook-learning.json
|   |   +-- webhook-community.json
|   |   +-- pull-wp-learning.json
|   |   +-- pull-wp-users.json
|   |   +-- pull-moyasar.json
|   |   +-- pull-bunny.json
|   |   +-- pull-resend.json
|   |   +-- pull-clickup.json
|   |   +-- pull-posthog.json
|   |   +-- orchestrator-nightly.json
|   +-- credentials.example.json
+-- scripts/                    # (as shown in Section 6.2)
+-- sql/
|   +-- schema.sql              # Full DDL (from dashboard spec + dead_letters)
|   +-- migrations/             # Versioned schema changes
+-- dashboard/                  # Next.js app (separate concern)
+-- .env.example
+-- package.json
```

### 8.2 Deployment

- **Scripts + dashboard:** deployed via git pull on VPS
- **n8n workflows:** import via n8n CLI (`n8n import:workflow --input=file.json`) or manual import through n8n UI
- **Schema changes:** versioned migration files in `sql/migrations/`, run manually

### 8.3 Credentials Management

- n8n credentials stored in n8n's internal DB (encrypted), not in git
- Scripts use `.env` file on VPS (API keys, Postgres connection string)
- `.env.example` in git with placeholder values

---

## 9. Monitoring

| What | How | Alert mechanism |
|------|-----|-----------------|
| Webhook processing | `cron_health` table, per-workflow status | Insight rule: "workflow X failed" |
| API pull completion | `cron_health` table, checked by gate.js | gate.js exit 1 triggers n8n error notification |
| Dead letters accumulation | Nightly rule: `SELECT COUNT(*) FROM dead_letters WHERE NOT resolved` | Insight: "N unresolved dead letters" |
| Data freshness | `pulse.updated_at` per section | Insight: "Revenue pulse stale > 24h" |
| Pipeline duration | `cron_health.duration_ms` | Insight if nightly run exceeds 60 min |
| Postgres health | `pg_stat_activity`, connection count | Script check in cleanup.js |

**Schema change required:** Parent spec's `cron_health` table has no UNIQUE constraint on `job_name`. Pipeline relies on `ON CONFLICT` upserts per workflow name. Add `UNIQUE(job_name)` constraint during implementation.

No external monitoring tool (PagerDuty, etc.) for V1. The dashboard itself IS the monitoring surface - the Operations section shows pipeline health. If something breaks overnight, you see it when you open the dashboard in the morning.

---

## 10. n8n Workflow Summary

**Total: 14 n8n workflows**

| Category | Count | Workflows |
|----------|-------|-----------|
| Webhook receivers | 6 | Revenue, Activity, Users, Security, Learning, Community |
| API pullers | 7 | WP Learning, WP Users, Moyasar, Bunny, Resend, ClickUp, PostHog |
| Orchestrator | 1 | Nightly pipeline (triggers gate -> compute -> insights -> cleanup) |

Each workflow self-reports to `cron_health` on completion. Webhook workflows run continuously (triggered by incoming POSTs). API pullers and orchestrator run on cron schedules.

---

## 11. Decision Log

| Decision | Options considered | Chosen | Rationale |
|----------|-------------------|--------|-----------|
| Error handling tier | No-loss-ever / Best-effort / Hybrid | Hybrid | Revenue+security need reliability; activity gaps are tolerable until nightly backfill |
| Workflow count | 23+ individual / ~9 domain-grouped / 2 mega-workflows | 14 domain-grouped | n8n's flat list doesn't scale to 23+; domain groups match dashboard sections and student ownership |
| Job ordering | Time-based / Chain-triggered / Hybrid gate | Hybrid gate | API pulls are independent (parallel OK); computations need fresh data (gate ensures it) |
| Idempotency | DB unique constraints / Explicit dedup queries / Event ID dedup | DB unique constraints | Most tables have natural keys; ON CONFLICT is native Postgres, zero extra infra |
| Transform location | All n8n / n8n + helper API / n8n Code nodes | All n8n | Transforms are simple enough; helper API adds deployment overhead not justified for ~10 functions |
| Computation location | All n8n / n8n + scripts / Separate worker service | n8n + scripts | SQL-heavy computations are painful in n8n's Code node; scripts are testable and version-controlled |
| Backfill | Start fresh / Full backfill / Selective backfill | Full backfill | Dashboard is useless without historical revenue and enrollment data |
| Data quality | Full validation framework / Lightweight checks / None | Lightweight checks | Validate critical fields per webhook type; dead_letters catch failures; not worth Great Expectations at this scale |
