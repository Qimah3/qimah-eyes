# Qimah Eyes Data Pipeline - Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the ingestion and ETL pipeline that feeds the Qimah Eyes analytics dashboard - webhook reception, API pulling, nightly computation, insight engine, and historical backfill.

**Architecture:** Three layers: (1) n8n ingests data via webhooks and nightly API pulls into Postgres, (2) Node.js scripts compute derived metrics (streaks, instructor scores, cohort stats, session patterns) in dependency order, (3) Postgres 16 stores everything. n8n moves data in, scripts compute, Postgres stores. 14 total n8n workflows (6 webhook + 7 API pull + 1 orchestrator).

**Tech Stack:** n8n (orchestration), Node.js 20+ (computation scripts), Postgres 16 (storage), pg npm package (DB client), Claude Sonnet API (weekly AI analysis)

**Specs:**
- Pipeline spec: `docs/superpowers/specs/2026-03-15-qimah-eyes-data-pipeline-design.md`
- Parent dashboard spec: `docs/superpowers/specs/2026-03-13-god-dashboard-design.md` (schema DDL, insight rules, dashboard UI)

**Environment:** Hetzner VPS (CPX41, 16GB RAM). n8n, Postgres 16, and Next.js already running (or will be deployed). Scripts deployed via git pull. n8n workflows built in n8n UI, exported as JSON to git.

**Dependencies:**
- WP-side analytics API (`GET /analytics/enrollments`, `GET /analytics/feedback`, `read:analytics` scope, `user.registered` webhook) - separate spec `2026-03-11-analytics-api-design.md`
- PostHog self-hosted on same VPS - deployed separately
- Existing WP webhooks (commerce, activity, session, points, streak, security, feedback, circle, booking) - most already wired in qimah-profile

---

## Chunk 1: Foundation (Repo + Schema + Shared Libs)

### Task 1: Initialize Repository

**Files:**
- Create: `qimah-eyes/package.json`
- Create: `qimah-eyes/.env.example`
- Create: `qimah-eyes/.gitignore`
- Create: `qimah-eyes/n8n/workflows/.gitkeep`
- Create: `qimah-eyes/sql/migrations/.gitkeep`

- [ ] **Step 1: Create repo directory and initialize npm**

```bash
mkdir -p qimah-eyes
cd qimah-eyes
npm init -y
```

- [ ] **Step 2: Install dependencies**

```bash
npm install pg dotenv
npm install -D vitest
```

- [ ] **Step 3: Create .env.example**

```env
# Postgres
DATABASE_URL=postgresql://qimah_eyes:password@localhost:5432/qimah_eyes

# API Keys (used by backfill.js)
WP_API_KEY=your-qimah-api-key
WP_BASE_URL=https://qimah.net/wp-json
WC_CONSUMER_KEY=your-woocommerce-consumer-key
WC_CONSUMER_SECRET=your-woocommerce-consumer-secret
MOYASAR_API_KEY=your-moyasar-api-key
BUNNY_API_KEY=your-bunny-api-key
BUNNY_LIBRARY_ID=your-bunny-library-id
RESEND_API_KEY=your-resend-api-key
CLICKUP_API_TOKEN=your-clickup-api-token
POSTHOG_API_KEY=your-posthog-personal-api-key
POSTHOG_HOST=http://localhost:8000

# Claude API (weekly AI analysis)
ANTHROPIC_API_KEY=your-anthropic-api-key

# Webhook HMAC validation (used by n8n, not scripts)
WEBHOOK_SECRET=your-webhook-hmac-secret
```

- [ ] **Step 4: Create .gitignore**

```
node_modules/
.env
*.log
```

- [ ] **Step 5: Create directory structure**

```bash
mkdir -p scripts/lib scripts/rules sql/migrations n8n/workflows
```

- [ ] **Step 6: Add scripts to package.json**

Add to `package.json`:
```json
{
  "scripts": {
    "gate": "node scripts/gate.js",
    "compute": "node scripts/compute.js",
    "insights": "node scripts/insights.js",
    "ai-weekly": "node scripts/ai-weekly.js",
    "cleanup": "node scripts/cleanup.js",
    "backfill": "node scripts/backfill.js",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

- [ ] **Step 7: Commit**

```bash
git init
git add -A
git commit -m "chore: initialize qimah-eyes repo with directory structure"
```

---

### Task 2: Write Full Schema DDL

**Files:**
- Create: `qimah-eyes/sql/schema.sql`

The full DDL comes from the parent dashboard spec (33 tables) plus pipeline-specific additions (dead_letters, work_tasks, schema modifications). All DDL is in one file for initial deployment.

- [ ] **Step 1: Write schema.sql**

Copy all CREATE TABLE statements from parent spec Sections 5.1-5.11, then add pipeline-specific tables and modifications:

```sql
-- Qimah Eyes - Full Schema
-- Source: 2026-03-13-god-dashboard-design.md (Sections 5.1-5.11)
-- Plus pipeline additions from 2026-03-15-qimah-eyes-data-pipeline-design.md

-- ============================================================
-- 5.1 PostHog Bridge
-- ============================================================

CREATE TABLE visitors (
  id              SERIAL PRIMARY KEY,
  anonymous_id    VARCHAR(100) UNIQUE NOT NULL,
  first_seen      TIMESTAMP,
  last_seen       TIMESTAMP,
  utm_source      VARCHAR(100),
  utm_medium      VARCHAR(100),
  utm_campaign    VARCHAR(200),
  referrer        TEXT,
  landing_page    TEXT,
  device_type     VARCHAR(20),
  country         VARCHAR(5),
  city            VARCHAR(100),
  pageviews       INT DEFAULT 0,
  created_at      TIMESTAMP DEFAULT NOW(),
  updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE funnels (
  id              SERIAL PRIMARY KEY,
  date            DATE NOT NULL,
  funnel_name     VARCHAR(50) NOT NULL,
  step            VARCHAR(50) NOT NULL,
  step_order      SMALLINT NOT NULL,
  count           INT NOT NULL,
  drop_off_pct    DECIMAL(5,2),
  UNIQUE(date, funnel_name, step)
);

CREATE TABLE conversions (
  id              SERIAL PRIMARY KEY,
  anonymous_id    VARCHAR(100),
  user_id         INT NOT NULL,
  registered_at   TIMESTAMP NOT NULL,
  first_purchase_at TIMESTAMP,
  attribution     JSONB,
  created_at      TIMESTAMP DEFAULT NOW(),
  updated_at      TIMESTAMP DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_conversions_user ON conversions(user_id);
CREATE INDEX idx_conversions_anon ON conversions(anonymous_id);

-- ============================================================
-- 5.2 Revenue Domain
-- ============================================================

CREATE TABLE customers (
  user_id           INT PRIMARY KEY,
  email             VARCHAR(255),
  display_name      VARCHAR(255),
  first_order_at    TIMESTAMPTZ,
  total_spent       DECIMAL(10,2) DEFAULT 0,
  order_count       INT DEFAULT 0,
  acquisition_source VARCHAR(50),
  first_coupon      VARCHAR(50),
  segment           VARCHAR(20) DEFAULT 'new',
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
  id                SERIAL PRIMARY KEY,
  wp_order_id       INT UNIQUE NOT NULL,
  user_id           INT NOT NULL,
  amount            DECIMAL(10,2) NOT NULL,
  currency          VARCHAR(3) DEFAULT 'SAR',
  payment_method    VARCHAR(30),
  payment_channel   VARCHAR(30),
  status            VARCHAR(20),
  coupon_code       VARCHAR(50),
  discount_pct      DECIMAL(5,2),
  created_at        TIMESTAMP NOT NULL,
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
  id                SERIAL PRIMARY KEY,
  order_id          INT NOT NULL REFERENCES orders(id),
  course_id         INT NOT NULL,
  course_title      VARCHAR(255),
  price             DECIMAL(10,2),
  discount_amount   DECIMAL(10,2) DEFAULT 0
);
CREATE INDEX idx_order_items_course ON order_items(course_id);

CREATE TABLE coupons (
  code              VARCHAR(50) PRIMARY KEY,
  type              VARCHAR(20),
  discount_value    DECIMAL(5,2),
  target_segment    VARCHAR(20),
  total_redemptions INT DEFAULT 0,
  total_revenue     DECIMAL(10,2) DEFAULT 0,
  avg_order_value   DECIMAL(10,2) DEFAULT 0,
  conversion_rate   DECIMAL(5,2),
  first_used        TIMESTAMP,
  last_used         TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE payment_failures (
  id                SERIAL PRIMARY KEY,
  moyasar_id        VARCHAR(100) UNIQUE,
  user_id           INT,
  wp_order_id       INT,
  method            VARCHAR(30),
  channel           VARCHAR(30),
  error_code        VARCHAR(50),
  retried           BOOLEAN DEFAULT false,
  retry_method      VARCHAR(30),
  recovered         BOOLEAN DEFAULT false,
  created_at        TIMESTAMP NOT NULL,
  updated_at        TIMESTAMP DEFAULT NOW()
);

-- ============================================================
-- 5.3 Learning Domain
-- ============================================================

CREATE TABLE course_snapshots (
  id                SERIAL PRIMARY KEY,
  course_id         INT NOT NULL,
  title             VARCHAR(255),
  instructor_id     INT,
  instructor_name   VARCHAR(255),
  enrolled_count    INT,
  completed_count   INT,
  avg_progress      DECIMAL(5,2),
  avg_feedback_pct  DECIMAL(5,2),
  snapshot_date     DATE NOT NULL,
  UNIQUE(course_id, snapshot_date),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE topic_stats (
  id                SERIAL PRIMARY KEY,
  course_id         INT NOT NULL,
  topic_id          INT NOT NULL,
  topic_title       VARCHAR(255),
  video_duration_sec INT,
  completions_7d    INT DEFAULT 0,
  avg_watch_pct     DECIMAL(5,2),
  feedback_up       INT DEFAULT 0,
  feedback_down     INT DEFAULT 0,
  snapshot_date     DATE NOT NULL,
  UNIQUE(topic_id, snapshot_date),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE instructor_scores (
  id                SERIAL PRIMARY KEY,
  instructor_id     INT NOT NULL,
  instructor_name   VARCHAR(255),
  course_id         INT NOT NULL,
  feedback_pct      DECIMAL(5,2),
  avg_video_length_sec INT,
  completion_rate   DECIMAL(5,2),
  trend_7d          DECIMAL(5,2),
  trend_30d         DECIMAL(5,2),
  snapshot_date     DATE NOT NULL,
  UNIQUE(instructor_id, course_id, snapshot_date)
);

CREATE TABLE feedback (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  course_id         INT NOT NULL,
  topic_id          INT,
  instructor_id     INT,
  vote              SMALLINT NOT NULL,
  comment           TEXT,
  created_at        TIMESTAMP NOT NULL
);
CREATE INDEX idx_feedback_course ON feedback(course_id, created_at);
CREATE INDEX idx_feedback_instructor ON feedback(instructor_id, created_at);

CREATE TABLE video_events (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  course_id         INT NOT NULL,
  topic_id          INT NOT NULL,
  video_id          VARCHAR(64),
  watch_duration_sec INT,
  total_duration_sec INT,
  watch_pct         DECIMAL(5,2),
  completed         BOOLEAN DEFAULT false,
  date              DATE,
  created_at        TIMESTAMP NOT NULL,
  UNIQUE(user_id, topic_id, date)
);
CREATE INDEX idx_video_events_topic ON video_events(topic_id, created_at);

-- ============================================================
-- 5.4 Retention Domain
-- ============================================================

CREATE TABLE user_profiles (
  user_id           INT PRIMARY KEY,
  registered_at     TIMESTAMPTZ,
  acquisition_source VARCHAR(50),
  first_course_id   INT,
  total_courses     INT DEFAULT 0,
  current_streak    INT DEFAULT 0,
  longest_streak    INT DEFAULT 0,
  last_active_at    TIMESTAMP,
  status            VARCHAR(20) DEFAULT 'active',
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE daily_activity (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  date              DATE NOT NULL,
  sessions          INT DEFAULT 0,
  topics_completed  INT DEFAULT 0,
  watch_minutes     DECIMAL(8,2) DEFAULT 0,
  points_earned     INT DEFAULT 0,
  UNIQUE(user_id, date),
  updated_at        TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_daily_activity_date ON daily_activity(date);

CREATE TABLE cohorts (
  id                SERIAL PRIMARY KEY,
  name              VARCHAR(100) NOT NULL,
  type              VARCHAR(20) NOT NULL,
  description       TEXT,
  user_count        INT DEFAULT 0,
  avg_retention_30d DECIMAL(5,2),
  churn_rate        DECIMAL(5,2),
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE cohort_members (
  cohort_id         INT NOT NULL REFERENCES cohorts(id),
  user_id           INT NOT NULL,
  joined_at         TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY(cohort_id, user_id)
);

CREATE TABLE streak_history (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  date              DATE NOT NULL,
  streak_count      INT NOT NULL,
  freeze_used       BOOLEAN DEFAULT false,
  UNIQUE(user_id, date)
);

-- ============================================================
-- 5.5 Security Domain
-- ============================================================

CREATE TABLE sharing_alerts (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  score             DECIMAL(5,2),
  flags             JSONB,
  status            VARCHAR(20) DEFAULT 'new',
  created_at        TIMESTAMP NOT NULL
);

CREATE TABLE session_patterns (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  devices_7d        INT,
  device_switches_7d INT,
  ip_count_7d       INT,
  anomaly_score     DECIMAL(5,2),
  snapshot_date     DATE NOT NULL,
  UNIQUE(user_id, snapshot_date)
);

-- ============================================================
-- 5.6 Community Domain
-- ============================================================

CREATE TABLE circles (
  id                SERIAL PRIMARY KEY,
  wp_circle_id      INT UNIQUE,
  name              VARCHAR(255),
  member_count      INT DEFAULT 0,
  challenge_type    VARCHAR(30),
  completion_rate   DECIMAL(5,2),
  created_at        TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE circle_activity (
  id                SERIAL PRIMARY KEY,
  circle_id         INT NOT NULL REFERENCES circles(id),
  date              DATE NOT NULL,
  messages          INT DEFAULT 0,
  active_members    INT DEFAULT 0,
  challenge_progress_pct DECIMAL(5,2),
  UNIQUE(circle_id, date)
);

CREATE TABLE discord_activity (
  id                SERIAL PRIMARY KEY,
  date              DATE NOT NULL UNIQUE,
  active_users      INT DEFAULT 0,
  messages          INT DEFAULT 0,
  voice_minutes     INT DEFAULT 0
);

-- ============================================================
-- 5.7 Operations Domain
-- ============================================================

CREATE TABLE video_pipeline (
  id                SERIAL PRIMARY KEY,
  date              DATE NOT NULL UNIQUE,
  uploads           INT DEFAULT 0,
  processed         INT DEFAULT 0,
  failed            INT DEFAULT 0,
  avg_process_time_sec INT,
  total_storage_gb  DECIMAL(8,2),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE cron_health (
  id                SERIAL PRIMARY KEY,
  job_name          VARCHAR(100) NOT NULL UNIQUE,
  last_run          TIMESTAMP,
  status            VARCHAR(20),
  duration_ms       INT,
  error_msg         TEXT,
  updated_at        TIMESTAMP DEFAULT NOW()
);

-- ============================================================
-- 5.8 Recruitment Domain
-- ============================================================

CREATE TABLE applicants (
  id                SERIAL PRIMARY KEY,
  clickup_id        VARCHAR(50) UNIQUE,
  name              VARCHAR(255),
  email             VARCHAR(255),
  track             VARCHAR(100),
  status            VARCHAR(50),
  applied_at        TIMESTAMP,
  reviewed_at       TIMESTAMP,
  hired_at          TIMESTAMP,
  ghosted           BOOLEAN DEFAULT false,
  had_prior_discord BOOLEAN DEFAULT false,
  source            VARCHAR(50),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE pipeline_snapshots (
  id                SERIAL PRIMARY KEY,
  track             VARCHAR(100) NOT NULL,
  status            VARCHAR(50) NOT NULL,
  count             INT NOT NULL,
  snapshot_date     DATE NOT NULL,
  UNIQUE(track, status, snapshot_date)
);

-- ============================================================
-- 5.9 Comms Domain
-- ============================================================

CREATE TABLE email_campaigns (
  id                SERIAL PRIMARY KEY,
  resend_id         VARCHAR(100) UNIQUE,
  subject           TEXT,
  target_segment    VARCHAR(50),
  sent_count        INT DEFAULT 0,
  open_rate         DECIMAL(5,2),
  click_rate        DECIMAL(5,2),
  bounce_count      INT DEFAULT 0,
  sent_at           TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE email_events (
  id                SERIAL PRIMARY KEY,
  resend_event_id   VARCHAR(100) UNIQUE,
  campaign_id       INT REFERENCES email_campaigns(id),
  user_id           INT,
  email             VARCHAR(255),
  event             VARCHAR(20),
  created_at        TIMESTAMP NOT NULL
);
CREATE INDEX idx_email_events_campaign ON email_events(campaign_id);

-- ============================================================
-- 5.10 Generic Event Log
-- ============================================================

CREATE TABLE events (
  id                SERIAL PRIMARY KEY,
  event_type        VARCHAR(100) NOT NULL,
  user_id           INT,
  payload           JSONB NOT NULL,
  source            VARCHAR(30),
  created_at        TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_user ON events(user_id, created_at);
CREATE INDEX idx_events_created ON events(created_at);

-- ============================================================
-- 5.11 Intelligence Layer
-- ============================================================

CREATE TABLE insights (
  id                SERIAL PRIMARY KEY,
  section           VARCHAR(20) NOT NULL,
  severity          VARCHAR(10) NOT NULL,
  headline          TEXT NOT NULL,
  body              TEXT NOT NULL,
  evidence          TEXT,
  actions           JSONB,
  rule_id           VARCHAR(50),
  is_ai             BOOLEAN DEFAULT false,
  is_pinned         BOOLEAN DEFAULT false,
  is_dismissed      BOOLEAN DEFAULT false,
  expires_at        TIMESTAMP,
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_insights_section ON insights(section, severity, created_at);
CREATE INDEX idx_insights_active ON insights(is_dismissed, expires_at);

CREATE TABLE pulse (
  section           VARCHAR(20) NOT NULL,
  key               VARCHAR(50) NOT NULL,
  value             NUMERIC,
  label             VARCHAR(100),
  delta             VARCHAR(50),
  sort_order        SMALLINT DEFAULT 0,
  updated_at        TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY(section, key)
);

CREATE TABLE summary (
  id                SERIAL PRIMARY KEY,
  date              DATE NOT NULL UNIQUE,
  action_count      INT DEFAULT 0,
  watch_count       INT DEFAULT 0,
  win_count         INT DEFAULT 0,
  computed_at       TIMESTAMP DEFAULT NOW()
);

-- ============================================================
-- Pipeline-specific tables (from pipeline spec)
-- ============================================================

-- Table #34: Dead letter queue for failed webhook/API pull events
CREATE TABLE dead_letters (
  id            SERIAL PRIMARY KEY,
  event_type    VARCHAR(100),
  payload       JSONB NOT NULL,
  error_message TEXT,
  source        VARCHAR(30),
  retries       INT DEFAULT 0,
  resolved      BOOLEAN DEFAULT false,
  created_at    TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_dead_letters_unresolved ON dead_letters(resolved, created_at) WHERE NOT resolved;

-- Table #35: ClickUp team task snapshots (new, not in parent spec)
CREATE TABLE work_tasks (
  id                SERIAL PRIMARY KEY,
  clickup_id        VARCHAR(50) UNIQUE NOT NULL,
  name              VARCHAR(500),
  status            VARCHAR(100),
  assignee_name     VARCHAR(255),
  list_name         VARCHAR(255),
  due_date          TIMESTAMP,
  date_created      TIMESTAMP,
  date_updated      TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT NOW()
);
```

- [ ] **Step 2: Verify SQL is valid**

```bash
# On VPS (or locally with Postgres):
psql -U qimah_eyes -d qimah_eyes -f sql/schema.sql
```

Expected: All tables created without errors. Verify with `\dt` showing 35 tables.

- [ ] **Step 3: Commit**

```bash
git add sql/schema.sql
git commit -m "feat: add full Postgres schema (35 tables)"
```

---

### Task 3: Implement Shared Libraries

**Files:**
- Create: `qimah-eyes/scripts/lib/db.js`
- Create: `qimah-eyes/scripts/lib/health.js`
- Create: `qimah-eyes/scripts/lib/__tests__/health.test.js`

- [ ] **Step 1: Write db.js**

```js
// scripts/lib/db.js
import pg from 'pg';
import { config } from 'dotenv';

config();

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 5,
  idleTimeoutMillis: 30000,
});

pool.on('error', (err) => {
  console.error('Unexpected pool error:', err.message);
  process.exit(1);
});

export default pool;
```

- [ ] **Step 2: Write health.js**

```js
// scripts/lib/health.js

/**
 * Report job status to cron_health table.
 * Upserts on job_name (UNIQUE constraint).
 *
 * @param {pg.Pool} db
 * @param {string} jobName
 * @param {'ok'|'failed'} status
 * @param {number} durationMs
 * @param {string|null} errorMsg
 */
export async function reportHealth(db, jobName, status, durationMs, errorMsg = null) {
  await db.query(
    `INSERT INTO cron_health (job_name, last_run, status, duration_ms, error_msg, updated_at)
     VALUES ($1, NOW(), $2, $3, $4, NOW())
     ON CONFLICT (job_name) DO UPDATE SET
       last_run = NOW(),
       status = $2,
       duration_ms = $3,
       error_msg = $4,
       updated_at = NOW()`,
    [jobName, status, durationMs, errorMsg]
  );
}
```

- [ ] **Step 3: Write test for health.js**

```js
// scripts/lib/__tests__/health.test.js
import { describe, it, expect, vi } from 'vitest';
import { reportHealth } from '../health.js';

describe('reportHealth', () => {
  it('calls db.query with correct upsert SQL and params', async () => {
    const db = { query: vi.fn().mockResolvedValue({ rowCount: 1 }) };

    await reportHealth(db, 'test_job', 'ok', 1234);

    expect(db.query).toHaveBeenCalledOnce();
    const [sql, params] = db.query.mock.calls[0];
    expect(sql).toContain('INSERT INTO cron_health');
    expect(sql).toContain('ON CONFLICT (job_name)');
    expect(params).toEqual(['test_job', 'ok', 1234, null]);
  });

  it('passes error message on failure', async () => {
    const db = { query: vi.fn().mockResolvedValue({ rowCount: 1 }) };

    await reportHealth(db, 'test_job', 'failed', 500, 'timeout');

    const [, params] = db.query.mock.calls[0];
    expect(params).toEqual(['test_job', 'failed', 500, 'timeout']);
  });
});
```

- [ ] **Step 4: Add type module to package.json**

Add `"type": "module"` to package.json for ES module imports.

- [ ] **Step 5: Run tests**

```bash
npx vitest run scripts/lib/__tests__/health.test.js
```

Expected: 2 tests pass.

- [ ] **Step 6: Commit**

```bash
git add scripts/lib/ package.json
git commit -m "feat: add shared db pool and cron_health reporter"
```

---

## Chunk 2: Webhook Ingestion (n8n Workflows)

n8n workflows are built in the n8n UI on the VPS, then exported as JSON to git. Each task describes the workflow structure, node types, and Code node JavaScript. Build in the n8n UI, test with a sample webhook, then export.

**Shared HMAC validation code** (used by all 6 webhook workflows in a Code node):

```js
// HMAC validation - first Code node in every webhook workflow
const crypto = require('crypto');
const secret = $env.WEBHOOK_SECRET;
const signature = $input.first().headers['x-qimah-signature'];
const body = JSON.stringify($input.first().json);
const expected = crypto.createHmac('sha256', secret).update(body).digest('hex');

// Constant-time comparison to prevent timing attacks
if (!signature || !crypto.timingSafeEqual(Buffer.from(signature, 'hex'), Buffer.from(expected, 'hex'))) {
  // Will be caught by error branch -> dead_letters
  throw new Error('HMAC validation failed');
}

return $input.all();
```

**Shared dead_letters insert** (used by error branches):

```sql
INSERT INTO dead_letters (event_type, payload, error_message, source, created_at)
VALUES ($1, $2::jsonb, $3, 'wp_webhook', NOW())
```

**Shared cron_health upsert** (last node in every workflow):

```sql
INSERT INTO cron_health (job_name, last_run, status, duration_ms, updated_at)
VALUES ($1, NOW(), 'ok', $2, NOW())
ON CONFLICT (job_name) DO UPDATE SET
  last_run = NOW(), status = 'ok', duration_ms = $2, updated_at = NOW()
```

### Task 4: Revenue Webhook Workflow

**Files:**
- Create (in n8n UI, export to): `qimah-eyes/n8n/workflows/webhook-revenue.json`

**Workflow nodes (build in n8n UI):**

```
Webhook (POST /webhook/qimah-revenue)
  -> Code: HMAC Validate
  -> Postgres: INSERT events (raw log)
  -> Code: Validate Shape
  -> Switch: event_type
     -> "commerce.order_paid":
        -> Code: Transform order
        -> Postgres: Upsert orders
        -> Postgres: Insert order_items
        -> Postgres: Upsert customers
        -> Postgres: Upsert coupons (if coupon_code present)
        -> Postgres: Upsert pulse (revenue_today)
     -> "commerce.order_refunded":
        -> Code: Transform refund
        -> Postgres: UPDATE orders SET status='refunded'
        -> Postgres: UPDATE customers (decrement totals)
        -> Postgres: Upsert pulse (refunds_today)
  -> Postgres: Upsert cron_health
  -> Error branch -> Postgres: INSERT dead_letters (with retry x3)
```

- [ ] **Step 1: Create workflow in n8n UI with Webhook trigger node**

Set path to `/webhook/qimah-revenue`, method POST, response mode "Last Node".

- [ ] **Step 2: Add HMAC validation Code node**

Use the shared HMAC validation code above. Connect error output to dead_letters Postgres node.

- [ ] **Step 3: Add raw event logging Postgres node**

Use n8n's Postgres node in **parameterized query mode** (not raw SQL interpolation) to prevent SQL injection:

- Column 1 (`event_type`): Expression `{{ $json.event_type }}`
- Column 2 (`user_id`): Expression `{{ $json.data.user_id || null }}`
- Column 3 (`payload`): Expression `{{ JSON.stringify($json) }}` (cast to jsonb)
- Column 4 (`source`): Fixed value `wp_webhook`
- Column 5 (`created_at`): `NOW()`

Equivalent SQL (parameterized):
```sql
INSERT INTO events (event_type, user_id, payload, source, created_at)
VALUES ($1, $2, $3::jsonb, 'wp_webhook', NOW())
```

- [ ] **Step 4: Add shape validation Code node**

```js
const data = $input.first().json.data;
const eventType = $input.first().json.event_type;

if (eventType === 'commerce.order_paid') {
  const required = ['order_id', 'total', 'user_id', 'items'];
  for (const field of required) {
    if (data[field] === undefined || data[field] === null) {
      throw new Error(`missing required field: ${field}`);
    }
  }
  if (typeof data.total !== 'number' || data.total <= 0) {
    throw new Error('invalid field: total must be number > 0');
  }
}

return $input.all();
```

- [ ] **Step 5: Add Switch node on event_type**

Conditions: `commerce.order_paid`, `commerce.order_refunded`.

- [ ] **Step 6: Add order upsert Postgres node (for order_paid branch)**

```sql
INSERT INTO orders (wp_order_id, user_id, amount, currency, payment_method, payment_channel, status, coupon_code, discount_pct, created_at, updated_at)
VALUES ($1, $2, $3, $4, $5, $6, 'paid', $7, $8, $9, NOW())
ON CONFLICT (wp_order_id) DO UPDATE SET
  status = 'paid',
  amount = EXCLUDED.amount,
  updated_at = NOW()
RETURNING id
```

Parameters: `order_id, user_id, total, currency, payment_method, payment_channel, coupon_code, discount_pct, created_at` (from webhook payload, with TZ conversion to UTC).

- [ ] **Step 7: Add order_items insert (loop over items array)**

Wire the `id` returned from the orders upsert node (`{{ $node['Upsert orders'].json.id }}`) as `order_id`. Use a SplitInBatches node or Code node to loop over `data.items`:

```sql
INSERT INTO order_items (order_id, course_id, course_title, price, discount_amount)
VALUES ($1, $2, $3, $4, $5)
```

Parameters: `$1` = `{{ $node['Upsert orders'].json.id }}`, `$2-$5` from each item.

- [ ] **Step 8: Add customer upsert**

```sql
INSERT INTO customers (user_id, email, display_name, first_order_at, total_spent, order_count, first_coupon, segment, created_at, updated_at)
VALUES ($1, $2, $3, NOW(), $4, 1, $5, 'new', NOW(), NOW())
ON CONFLICT (user_id) DO UPDATE SET
  total_spent = customers.total_spent + EXCLUDED.total_spent,
  order_count = customers.order_count + 1,
  segment = CASE WHEN customers.order_count >= 3 THEN 'vip' WHEN customers.order_count >= 2 THEN 'returning' ELSE 'new' END,
  updated_at = NOW()
```

- [ ] **Step 9: Add coupon stats upsert (conditional - only if coupon_code present)**

```sql
INSERT INTO coupons (code, total_redemptions, total_revenue, avg_order_value, last_used, updated_at)
VALUES ($1, 1, $2, $2, NOW(), NOW())
ON CONFLICT (code) DO UPDATE SET
  total_redemptions = coupons.total_redemptions + 1,
  total_revenue = coupons.total_revenue + $2,
  avg_order_value = (coupons.total_revenue + $2) / (coupons.total_redemptions + 1),
  last_used = NOW(),
  updated_at = NOW()
```

- [ ] **Step 10: Add pulse upsert**

```sql
INSERT INTO pulse (section, key, value, label, sort_order, updated_at)
VALUES ('revenue', 'revenue_today', $1, 'Revenue Today (SAR)', 1, NOW())
ON CONFLICT (section, key) DO UPDATE SET
  value = pulse.value + $1,
  updated_at = NOW()
```

- [ ] **Step 11: Add refund branch (commerce.order_refunded)**

Update order status and decrement customer totals:

```sql
UPDATE orders SET status = 'refunded', updated_at = NOW()
WHERE wp_order_id = $1
```

```sql
UPDATE customers SET
  total_spent = total_spent - $1,
  order_count = GREATEST(order_count - 1, 0),
  updated_at = NOW()
WHERE user_id = $2
```

- [ ] **Step 12: Add error handling branch with dead_letters insert + 3x retry**

Configure n8n error workflow with retry: 3 attempts, backoff 1min/5min/15min.

- [ ] **Step 13: Add cron_health upsert as final node**

Use shared cron_health SQL above with `job_name = 'webhook_revenue'`.

- [ ] **Step 14: Test with sample webhook payload**

Send a test POST to the webhook URL with a sample `commerce.order_paid` payload. Verify: event logged, order inserted, customer upserted, pulse updated, cron_health updated.

- [ ] **Step 15: Export workflow JSON and commit**

```bash
# In n8n UI: Workflow menu -> Export -> Download as JSON
# Save to n8n/workflows/webhook-revenue.json
git add n8n/workflows/webhook-revenue.json
git commit -m "feat: add revenue webhook workflow"
```

---

### Task 5: Activity & Sessions Webhook Workflow

**Files:**
- Create (in n8n UI, export to): `qimah-eyes/n8n/workflows/webhook-activity.json`

**Workflow nodes:**

```
Webhook (POST /webhook/qimah-activity)
  -> Code: HMAC Validate
  -> Postgres: INSERT events
  -> Switch: event_type
     -> "activity.logged" / "topic.completed" / "session.started":
        -> Code: Transform (TZ to UTC, extract date)
        -> Postgres: Upsert daily_activity
     -> "points.awarded":
        -> Postgres: UPDATE daily_activity SET points_earned = points_earned + $1
     -> "activity.streak_milestone":
        -> Postgres: Upsert streak_history
  -> Postgres: Upsert pulse (sessions_today, topics_today)
  -> Postgres: Upsert cron_health
  -> Error branch -> Postgres: INSERT dead_letters (no retry - best-effort)
```

- [ ] **Step 1: Create workflow with Webhook trigger**

Path: `/webhook/qimah-activity`, method POST.

- [ ] **Step 2: Add HMAC validation + raw event logging (same pattern as Revenue)**

- [ ] **Step 3: Add Switch node for 5 event types**

Routes: `activity.logged`, `topic.completed`, `session.started`, `points.awarded`, `activity.streak_milestone`.

- [ ] **Step 4: Add daily_activity upsert Code + Postgres node**

Transform code (extracts date from timestamp):
```js
const data = $input.first().json.data;
// Convert AST timestamp to UTC, extract date
const ts = new Date(data.timestamp);
const date = ts.toISOString().split('T')[0];
const userId = data.user_id;
const eventType = $input.first().json.event_type;

return [{
  json: {
    user_id: userId,
    date,
    is_session: eventType === 'session.started' ? 1 : 0,
    is_topic: eventType === 'topic.completed' ? 1 : 0,
  }
}];
```

Postgres upsert:
```sql
INSERT INTO daily_activity (user_id, date, sessions, topics_completed, updated_at)
VALUES ($1, $2, $3, $4, NOW())
ON CONFLICT (user_id, date) DO UPDATE SET
  sessions = daily_activity.sessions + $3,
  topics_completed = daily_activity.topics_completed + $4,
  updated_at = NOW()
```

- [ ] **Step 5: Add points_earned update for points.awarded**

```sql
INSERT INTO daily_activity (user_id, date, points_earned, updated_at)
VALUES ($1, $2::date, $3, NOW())
ON CONFLICT (user_id, date) DO UPDATE SET
  points_earned = daily_activity.points_earned + EXCLUDED.points_earned,
  updated_at = NOW()
```

- [ ] **Step 6: Add streak_history upsert for streak_milestone**

```sql
INSERT INTO streak_history (user_id, date, streak_count, freeze_used)
VALUES ($1, $2, $3, $4)
ON CONFLICT (user_id, date) DO UPDATE SET
  streak_count = EXCLUDED.streak_count,
  freeze_used = EXCLUDED.freeze_used
```

- [ ] **Step 7: Add pulse updates + cron_health + error branch (no retry)**

- [ ] **Step 8: Test, export, commit**

```bash
git add n8n/workflows/webhook-activity.json
git commit -m "feat: add activity & sessions webhook workflow"
```

---

### Task 6: Users Webhook Workflow

**Files:**
- Create: `qimah-eyes/n8n/workflows/webhook-users.json`

**Workflow nodes:**

```
Webhook (POST /webhook/qimah-users)
  -> Code: HMAC Validate
  -> Postgres: INSERT events
  -> Code: Transform user data
  -> Postgres: Upsert user_profiles
  -> Postgres: Upsert conversions
  -> HTTP Request: PostHog identify (merge anonymous_id with user_id)
  -> Postgres: Upsert pulse (registrations_today)
  -> Postgres: Upsert cron_health
  -> Error branch -> dead_letters
```

- [ ] **Step 1: Create workflow with Webhook trigger, HMAC, event logging**

- [ ] **Step 2: Add user_profiles upsert**

```sql
INSERT INTO user_profiles (user_id, registered_at, status, updated_at)
VALUES ($1, $2, 'active', NOW())
ON CONFLICT (user_id) DO UPDATE SET updated_at = NOW()
```

- [ ] **Step 3: Add conversions upsert**

```sql
INSERT INTO conversions (user_id, registered_at, attribution, created_at, updated_at)
VALUES ($1, $2, $3::jsonb, NOW(), NOW())
ON CONFLICT (user_id) DO UPDATE SET
  attribution = COALESCE(EXCLUDED.attribution, conversions.attribution),
  updated_at = NOW()
```

Attribution JSON extracted from webhook payload UTM data if available.

- [ ] **Step 4: Add PostHog identify HTTP Request node**

```
POST {{ $env.POSTHOG_HOST }}/api/event/
Headers: Content-Type: application/json
Body:
{
  "api_key": "{{ $env.POSTHOG_API_KEY }}",
  "event": "$identify",
  "distinct_id": "{{ $json.data.user_id }}",
  "$set": {
    "email": "{{ $json.data.email }}",
    "name": "{{ $json.data.display_name }}"
  },
  "properties": {
    "$anon_distinct_id": "{{ $json.data.anonymous_id }}"
  }
}
```

- [ ] **Step 5: Add pulse, cron_health, error branch. Test, export, commit.**

```bash
git add n8n/workflows/webhook-users.json
git commit -m "feat: add users webhook workflow"
```

---

### Task 7: Security Webhook Workflow

**Files:**
- Create: `qimah-eyes/n8n/workflows/webhook-security.json`

**Workflow nodes:**

```
Webhook (POST /webhook/qimah-security)
  -> Code: HMAC Validate
  -> Postgres: INSERT events (raw log - this is the primary data for session_patterns computation)
  -> Switch: event_type
     -> "sharing.*":
        -> Postgres: INSERT sharing_alerts
     -> "device.*" / "session.*":
        -> (no domain write - raw events table is sufficient, session_patterns computed nightly)
  -> Postgres: Upsert pulse (security section)
  -> Postgres: Upsert cron_health
  -> Error branch -> dead_letters (WITH retry x3 - DLQ-retried tier)
```

- [ ] **Step 1: Create workflow. This is a DLQ-retried tier - configure n8n retry: 3 attempts, backoff 1min/5min/15min.**

- [ ] **Step 2: Add sharing_alerts insert**

```sql
INSERT INTO sharing_alerts (user_id, score, flags, status, created_at)
VALUES ($1, $2, $3::jsonb, 'new', $4)
```

- [ ] **Step 3: Add pulse, cron_health, error branch with retry. Test, export, commit.**

```bash
git add n8n/workflows/webhook-security.json
git commit -m "feat: add security webhook workflow (DLQ-retried)"
```

---

### Task 8: Learning Webhook Workflow

**Files:**
- Create: `qimah-eyes/n8n/workflows/webhook-learning.json`

**Workflow nodes:**

```
Webhook (POST /webhook/qimah-learning)
  -> Code: HMAC Validate
  -> Postgres: INSERT events
  -> Code: Transform feedback (rating string -> vote int, derive instructor_id)
  -> Postgres: INSERT feedback
  -> Postgres: Upsert pulse
  -> Postgres: Upsert cron_health
  -> Error branch -> dead_letters (no retry)
```

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Add feedback transform Code node**

```js
const data = $input.first().json.data;

// Transform rating to vote int
let vote;
if (data.rating === 'up' || data.rating === 1 || data.rating === true) {
  vote = 1;
} else {
  vote = -1;
}

// instructor_id from webhook payload (derived from course author on WP side)
return [{
  json: {
    user_id: data.user_id,
    course_id: data.course_id,
    topic_id: data.topic_id || null,
    instructor_id: data.instructor_id || null,
    vote,
    comment: data.comment || null,
    created_at: new Date(data.timestamp).toISOString(),
  }
}];
```

- [ ] **Step 3: Add feedback INSERT, pulse, cron_health, error. Test, export, commit.**

```bash
git add n8n/workflows/webhook-learning.json
git commit -m "feat: add learning webhook workflow"
```

---

### Task 9: Community & Bookings Webhook Workflow

**Files:**
- Create: `qimah-eyes/n8n/workflows/webhook-community.json`

**Workflow nodes:**

```
Webhook (POST /webhook/qimah-community)
  -> Code: HMAC Validate
  -> Postgres: INSERT events
  -> Switch: event_type prefix
     -> "circle.*":
        -> Postgres: Upsert circles
        -> Postgres: Upsert circle_activity
     -> "booking.*":
        -> (no domain write - logged to events only for V1)
  -> Postgres: Upsert pulse
  -> Postgres: Upsert cron_health
  -> Error branch -> dead_letters (no retry)
```

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Add circles upsert**

```sql
INSERT INTO circles (wp_circle_id, name, member_count, challenge_type, created_at, updated_at)
VALUES ($1, $2, $3, $4, $5, NOW())
ON CONFLICT (wp_circle_id) DO UPDATE SET
  name = EXCLUDED.name,
  member_count = EXCLUDED.member_count,
  updated_at = NOW()
```

- [ ] **Step 3: Add circle_activity upsert**

```sql
INSERT INTO circle_activity (circle_id, date, messages, active_members, challenge_progress_pct)
VALUES (
  (SELECT id FROM circles WHERE wp_circle_id = $1),
  $2, $3, $4, $5
)
ON CONFLICT (circle_id, date) DO UPDATE SET
  messages = circle_activity.messages + EXCLUDED.messages,
  active_members = GREATEST(circle_activity.active_members, EXCLUDED.active_members)
```

- [ ] **Step 4: Add pulse, cron_health, error. Test, export, commit.**

```bash
git add n8n/workflows/webhook-community.json
git commit -m "feat: add community & bookings webhook workflow"
```

---

## Chunk 3: API Pull Ingestion (n8n Workflows)

7 nightly cron-triggered workflows. Each follows the same pattern: paginate API, transform in Code node, upsert to domain tables, report to cron_health.

### Task 10: WP Learning Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-wp-learning.json`

**Schedule:** 00:00 UTC daily.

**Workflow nodes:**

```
Cron (00:00 UTC)
  -> HTTP Request: GET /analytics/enrollments (paginated)
  -> Code: Transform enrollments -> course_snapshots format
  -> Postgres: Upsert course_snapshots
  -> HTTP Request: GET /analytics/feedback (paginated)
  -> Code: Transform feedback -> update course_snapshots.avg_feedback_pct
  -> Postgres: UPDATE course_snapshots
  -> HTTP Request: GET /wp/v2/sfwd-courses?_fields=id,author (paginated)
  -> Code: Map course author -> instructor_id
  -> Postgres: UPDATE course_snapshots SET instructor_id, instructor_name
  -> Postgres: Upsert cron_health ('pull_wp_learning', 'ok')
  -> Error branch -> dead_letters + cron_health ('pull_wp_learning', 'failed')
```

- [ ] **Step 1: Create workflow with Schedule trigger (00:00 UTC)**

- [ ] **Step 2: Add paginated enrollment fetch**

HTTP Request node with pagination: cursor-based, loop until no more results.
URL: `{{ $env.WP_BASE_URL }}/qimah/v1/analytics/enrollments`
Headers: `X-Qimah-API-Key: {{ $env.WP_API_KEY }}`

- [ ] **Step 3: Add course_snapshots upsert**

```sql
INSERT INTO course_snapshots (course_id, title, enrolled_count, completed_count, avg_progress, snapshot_date, updated_at)
VALUES ($1, $2, $3, $4, $5, CURRENT_DATE, NOW())
ON CONFLICT (course_id, snapshot_date) DO UPDATE SET
  enrolled_count = EXCLUDED.enrolled_count,
  completed_count = EXCLUDED.completed_count,
  avg_progress = EXCLUDED.avg_progress,
  updated_at = NOW()
```

- [ ] **Step 4: Add feedback fetch and avg_feedback_pct update**

- [ ] **Step 5: Add course author mapping for instructor_id**

- [ ] **Step 6: Add topic_stats population from enrollment data**

```sql
INSERT INTO topic_stats (course_id, topic_id, topic_title, completions_7d, snapshot_date, updated_at)
VALUES ($1, $2, $3, $4, CURRENT_DATE, NOW())
ON CONFLICT (topic_id, snapshot_date) DO UPDATE SET
  completions_7d = EXCLUDED.completions_7d,
  updated_at = NOW()
```

- [ ] **Step 7: Add cron_health + error branch. Test, export, commit.**

```bash
git add n8n/workflows/pull-wp-learning.json
git commit -m "feat: add WP learning nightly pull workflow"
```

---

### Task 11: WP Users Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-wp-users.json`

**Schedule:** 00:00 UTC daily.

- [ ] **Step 1: Create workflow with Schedule trigger**

- [ ] **Step 2: Add paginated leaderboard fetch**

URL: `{{ $env.WP_BASE_URL }}/qimah/v1/leaderboard`

- [ ] **Step 3: Add user_profiles upsert**

```sql
INSERT INTO user_profiles (user_id, current_streak, longest_streak, last_active_at, total_courses, updated_at)
VALUES ($1, $2, $3, $4, $5, NOW())
ON CONFLICT (user_id) DO UPDATE SET
  current_streak = EXCLUDED.current_streak,
  longest_streak = EXCLUDED.longest_streak,
  last_active_at = EXCLUDED.last_active_at,
  total_courses = EXCLUDED.total_courses,
  updated_at = NOW()
```

- [ ] **Step 4: Add cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-wp-users.json
git commit -m "feat: add WP users nightly pull workflow"
```

---

### Task 12: Moyasar Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-moyasar.json`

**Schedule:** 00:00 UTC daily.

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Fetch failed payments from Moyasar API**

URL: `https://api.moyasar.com/v1/payments?status=failed&created_after={{ yesterday }}`
Auth: Bearer `{{ $env.MOYASAR_API_KEY }}`

- [ ] **Step 3: Upsert payment_failures**

```sql
INSERT INTO payment_failures (moyasar_id, user_id, wp_order_id, method, channel, error_code, created_at, updated_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
ON CONFLICT (moyasar_id) DO UPDATE SET
  updated_at = NOW()
```

- [ ] **Step 4: cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-moyasar.json
git commit -m "feat: add Moyasar nightly pull workflow"
```

---

### Task 13: Bunny Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-bunny.json`

**Schedule:** 00:00 UTC daily.

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Fetch video stats from Bunny Stream API + WP video-manager**

- [ ] **Step 3: Upsert video_pipeline (daily stats)**

```sql
INSERT INTO video_pipeline (date, uploads, processed, failed, avg_process_time_sec, total_storage_gb, updated_at)
VALUES (CURRENT_DATE, $1, $2, $3, $4, $5, NOW())
ON CONFLICT (date) DO UPDATE SET
  uploads = EXCLUDED.uploads,
  processed = EXCLUDED.processed,
  failed = EXCLUDED.failed,
  updated_at = NOW()
```

- [ ] **Step 4: Update topic_stats with video durations**

```sql
UPDATE topic_stats SET
  video_duration_sec = $1,
  updated_at = NOW()
WHERE topic_id = $2 AND snapshot_date = CURRENT_DATE
```

- [ ] **Step 5: cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-bunny.json
git commit -m "feat: add Bunny CDN nightly pull workflow"
```

---

### Task 14: Resend Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-resend.json`

**Schedule:** 00:00 UTC daily.

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Fetch emails/domains from Resend API**

- [ ] **Step 3: Upsert email_campaigns**

```sql
INSERT INTO email_campaigns (resend_id, subject, sent_count, open_rate, click_rate, bounce_count, sent_at, updated_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
ON CONFLICT (resend_id) DO UPDATE SET
  sent_count = EXCLUDED.sent_count,
  open_rate = EXCLUDED.open_rate,
  click_rate = EXCLUDED.click_rate,
  bounce_count = EXCLUDED.bounce_count,
  updated_at = NOW()
```

- [ ] **Step 4: Upsert email_events (deduped by resend_event_id)**

```sql
INSERT INTO email_events (resend_event_id, campaign_id, user_id, email, event, created_at)
VALUES ($1, $2, $3, $4, $5, $6)
ON CONFLICT (resend_event_id) DO NOTHING
```

- [ ] **Step 5: cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-resend.json
git commit -m "feat: add Resend nightly pull workflow"
```

---

### Task 15: ClickUp Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-clickup.json`

**Schedule:** 00:00 UTC daily.

- [ ] **Step 1: Create workflow**

- [ ] **Step 2: Fetch tasks from ClickUp API (team tasks + instructor pipeline)**

- [ ] **Step 3: Upsert work_tasks**

```sql
INSERT INTO work_tasks (clickup_id, name, status, assignee_name, list_name, due_date, date_created, date_updated, updated_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
ON CONFLICT (clickup_id) DO UPDATE SET
  name = EXCLUDED.name,
  status = EXCLUDED.status,
  assignee_name = EXCLUDED.assignee_name,
  list_name = EXCLUDED.list_name,
  due_date = EXCLUDED.due_date,
  date_updated = EXCLUDED.date_updated,
  updated_at = NOW()
```

- [ ] **Step 4: Upsert applicants (instructor pipeline tasks)**

```sql
INSERT INTO applicants (clickup_id, name, email, track, status, applied_at, updated_at)
VALUES ($1, $2, $3, $4, $5, $6, NOW())
ON CONFLICT (clickup_id) DO UPDATE SET
  status = EXCLUDED.status,
  name = EXCLUDED.name,
  updated_at = NOW()
```

- [ ] **Step 5: Insert pipeline_snapshots (daily snapshot of pipeline state)**

```sql
INSERT INTO pipeline_snapshots (track, status, count, snapshot_date)
VALUES ($1, $2, $3, CURRENT_DATE)
ON CONFLICT (track, status, snapshot_date) DO UPDATE SET
  count = EXCLUDED.count
```

- [ ] **Step 6: cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-clickup.json
git commit -m "feat: add ClickUp nightly pull workflow"
```

---

### Task 16: PostHog Pull

**Files:**
- Create: `qimah-eyes/n8n/workflows/pull-posthog.json`

**Schedule:** 00:30 UTC daily (delayed for PostHog's own nightly aggregation).

- [ ] **Step 1: Create workflow with Schedule trigger at 00:30 UTC**

- [ ] **Step 2: Fetch visitors from PostHog persons API**

```
GET {{ $env.POSTHOG_HOST }}/api/projects/@current/persons/?properties=[{"key":"$initial_utm_source","type":"person"}]
```

Transform and upsert to `visitors` table.

- [ ] **Step 3: Fetch funnel data from PostHog insights API**

Pre-defined funnel IDs for visit_to_purchase and register_to_active. Upsert to `funnels` table.

- [ ] **Step 4: Fetch conversions (merge anonymous -> user_id)**

Query PostHog for identified users, upsert to `conversions` table.

- [ ] **Step 5: Fetch video events (Bunny iframe autocapture)**

Transform PostHog autocapture events into aggregated video_events:

```js
// Code node: aggregate video events by user+topic+date
const events = $input.all().map(i => i.json);
const grouped = {};

for (const e of events) {
  const key = `${e.distinct_id}_${e.properties.topic_id}_${e.properties.$session_date || e.timestamp.split('T')[0]}`;
  if (!grouped[key]) {
    grouped[key] = {
      user_id: parseInt(e.distinct_id),
      topic_id: e.properties.topic_id,
      course_id: e.properties.course_id,
      date: e.timestamp.split('T')[0],
      watch_duration_sec: 0,
      max_pct: 0,
      play_count: 0,
    };
  }
  grouped[key].watch_duration_sec += e.properties.duration || 0;
  grouped[key].max_pct = Math.max(grouped[key].max_pct, e.properties.percent_watched || 0);
  grouped[key].play_count++;
}

return Object.values(grouped).map(g => ({ json: g }));
```

Upsert:
```sql
INSERT INTO video_events (user_id, course_id, topic_id, watch_duration_sec, watch_pct, date, created_at)
VALUES ($1, $2, $3, $4, $5, $6, NOW())
ON CONFLICT (user_id, topic_id, date) DO UPDATE SET
  watch_duration_sec = EXCLUDED.watch_duration_sec,
  watch_pct = EXCLUDED.watch_pct,
  created_at = NOW()
```

- [ ] **Step 6: cron_health + error. Test, export, commit.**

```bash
git add n8n/workflows/pull-posthog.json
git commit -m "feat: add PostHog nightly pull workflow (00:30 UTC)"
```

---

## Chunk 4: Computation Pipeline (Node.js Scripts)

### Task 17: Gate Script

**Files:**
- Create: `qimah-eyes/scripts/gate.js`
- Create: `qimah-eyes/scripts/__tests__/gate.test.js`

- [ ] **Step 1: Write test for gate logic**

```js
// scripts/__tests__/gate.test.js
import { describe, it, expect, vi } from 'vitest';
import { checkGate } from '../gate.js';

const REQUIRED = ['pull_wp_learning', 'pull_moyasar'];
const ALL_JOBS = ['pull_wp_learning', 'pull_wp_users', 'pull_moyasar', 'pull_bunny', 'pull_resend', 'pull_clickup', 'pull_posthog'];

describe('checkGate', () => {
  it('returns true when all jobs are ok', async () => {
    const db = {
      query: vi.fn().mockResolvedValue({
        rows: ALL_JOBS.map(j => ({ job_name: j, status: 'ok' }))
      })
    };
    const result = await checkGate(db, ALL_JOBS, REQUIRED);
    expect(result).toBe(true);
  });

  it('returns false when a required job failed', async () => {
    const db = {
      query: vi.fn().mockResolvedValue({
        rows: ALL_JOBS.map(j => ({
          job_name: j,
          status: j === 'pull_wp_learning' ? 'failed' : 'ok'
        }))
      })
    };
    const result = await checkGate(db, ALL_JOBS, REQUIRED);
    expect(result).toBe(false);
  });

  it('returns true when only optional jobs failed', async () => {
    const db = {
      query: vi.fn().mockResolvedValue({
        rows: ALL_JOBS.map(j => ({
          job_name: j,
          status: j === 'pull_bunny' ? 'failed' : 'ok'
        }))
      })
    };
    const result = await checkGate(db, ALL_JOBS, REQUIRED);
    expect(result).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx vitest run scripts/__tests__/gate.test.js
```

Expected: FAIL - `checkGate` not defined.

- [ ] **Step 3: Write gate.js**

```js
// scripts/gate.js
import db from './lib/db.js';

const ALL_JOBS = [
  'pull_wp_learning', 'pull_wp_users', 'pull_moyasar',
  'pull_bunny', 'pull_resend', 'pull_clickup', 'pull_posthog'
];
const REQUIRED = ['pull_wp_learning', 'pull_moyasar'];
const POLL_INTERVAL_MS = 60_000;
const TIMEOUT_MS = 30 * 60_000;

export async function checkGate(pool, allJobs, requiredJobs) {
  const { rows } = await pool.query(
    `SELECT job_name, status FROM cron_health
     WHERE job_name = ANY($1)
       AND last_run >= CURRENT_DATE`,
    [allJobs]
  );

  const statusMap = new Map(rows.map(r => [r.job_name, r.status]));

  // Check if all jobs have reported
  const allReported = allJobs.every(j => statusMap.has(j));
  if (!allReported) return null; // Not all done yet

  // Check required jobs
  for (const job of requiredJobs) {
    if (statusMap.get(job) === 'failed') return false;
  }

  return true;
}

// Main: poll until gate passes or timeout
async function main() {
  const start = Date.now();
  console.log('Gate: waiting for API pulls to complete...');

  while (Date.now() - start < TIMEOUT_MS) {
    const result = await checkGate(db, ALL_JOBS, REQUIRED);

    if (result === true) {
      console.log('Gate: all required pulls completed. Proceeding.');
      await db.end();
      process.exit(0);
    }

    if (result === false) {
      console.error('Gate: required pull failed. Aborting.');
      await db.end();
      process.exit(1);
    }

    // null = not all reported yet, keep polling
    console.log(`Gate: waiting... (${Math.round((Date.now() - start) / 1000)}s elapsed)`);
    await new Promise(r => setTimeout(r, POLL_INTERVAL_MS));
  }

  console.error('Gate: timeout after 30 minutes. Aborting.');
  await db.end();
  process.exit(1);
}

// Only run main if executed directly (not imported for testing)
if (process.argv[1]?.endsWith('gate.js')) {
  main();
}
```

- [ ] **Step 4: Run tests**

```bash
npx vitest run scripts/__tests__/gate.test.js
```

Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/gate.js scripts/__tests__/gate.test.js
git commit -m "feat: add gate script (polls cron_health for API pull completion)"
```

---

### Task 18: Compute Script

**Files:**
- Create: `qimah-eyes/scripts/compute.js`
- Create: `qimah-eyes/scripts/__tests__/compute.test.js`

- [ ] **Step 1: Write test for compute runner pattern**

```js
// scripts/__tests__/compute.test.js
import { describe, it, expect, vi } from 'vitest';
import { runSteps } from '../compute.js';

describe('runSteps', () => {
  it('runs all steps in order and reports health', async () => {
    const order = [];
    const db = { query: vi.fn().mockResolvedValue({ rows: [] }) };
    const reportHealth = vi.fn();
    const steps = [
      { name: 'step_a', fn: async () => { order.push('a'); } },
      { name: 'step_b', fn: async () => { order.push('b'); } },
    ];

    await runSteps(db, steps, reportHealth);

    expect(order).toEqual(['a', 'b']);
    expect(reportHealth).toHaveBeenCalledTimes(2);
    expect(reportHealth.mock.calls[0][1]).toBe('step_a');
    expect(reportHealth.mock.calls[0][2]).toBe('ok');
  });

  it('stops on failure and reports failed status', async () => {
    const db = { query: vi.fn() };
    const reportHealth = vi.fn();
    const steps = [
      { name: 'step_a', fn: async () => {} },
      { name: 'step_b', fn: async () => { throw new Error('boom'); } },
      { name: 'step_c', fn: async () => {} },
    ];

    await expect(runSteps(db, steps, reportHealth)).rejects.toThrow('boom');

    expect(reportHealth).toHaveBeenCalledTimes(2);
    expect(reportHealth.mock.calls[1][2]).toBe('failed');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Write compute.js**

```js
// scripts/compute.js
import db from './lib/db.js';
import { reportHealth } from './lib/health.js';

export async function runSteps(pool, steps, healthReporter) {
  for (const step of steps) {
    const start = Date.now();
    try {
      await step.fn(pool);
      await healthReporter(pool, step.name, 'ok', Date.now() - start);
    } catch (err) {
      await healthReporter(pool, step.name, 'failed', Date.now() - start, err.message);
      throw err;
    }
  }
}

// --- Computation functions (each is a single SQL query) ---

async function computeStreaks(pool) {
  await pool.query(`
    WITH islands AS (
      SELECT user_id, date,
             date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date))::int AS island_id
      FROM daily_activity
      WHERE date >= CURRENT_DATE - INTERVAL '90 days'
        AND (sessions > 0 OR topics_completed > 0)
    ),
    streaks AS (
      SELECT user_id, date,
             ROW_NUMBER() OVER (PARTITION BY user_id, island_id ORDER BY date)::int AS streak_count
      FROM islands
    )
    INSERT INTO streak_history (user_id, date, streak_count)
    SELECT user_id, date, streak_count FROM streaks
    ON CONFLICT (user_id, date) DO UPDATE SET
      streak_count = EXCLUDED.streak_count
  `);
}

async function updateUserStreaks(pool) {
  await pool.query(`
    UPDATE user_profiles up SET
      current_streak = COALESCE(sh.streak_count, 0),
      longest_streak = GREATEST(up.longest_streak, COALESCE(sh.streak_count, 0)),
      updated_at = NOW()
    FROM (
      SELECT user_id, streak_count
      FROM streak_history
      WHERE date = CURRENT_DATE - 1
    ) sh
    WHERE up.user_id = sh.user_id
  `);
}

async function computeInstructorScores(pool) {
  await pool.query(`
    INSERT INTO instructor_scores (instructor_id, instructor_name, course_id, feedback_pct, avg_video_length_sec, completion_rate, trend_7d, trend_30d, snapshot_date)
    SELECT
      cs.instructor_id,
      cs.instructor_name,
      cs.course_id,
      CASE WHEN fb.total > 0 THEN (fb.ups::decimal / fb.total) * 100 ELSE NULL END AS feedback_pct,
      ts_avg.avg_duration AS avg_video_length_sec,
      CASE WHEN cs.enrolled_count > 0 THEN (cs.completed_count::decimal / cs.enrolled_count) * 100 ELSE NULL END AS completion_rate,
      NULL AS trend_7d,
      NULL AS trend_30d,
      CURRENT_DATE AS snapshot_date
    FROM course_snapshots cs
    LEFT JOIN (
      SELECT instructor_id, course_id,
             COUNT(*) AS total,
             COUNT(*) FILTER (WHERE vote = 1) AS ups
      FROM feedback
      WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
      GROUP BY instructor_id, course_id
    ) fb ON fb.instructor_id = cs.instructor_id AND fb.course_id = cs.course_id
    LEFT JOIN (
      SELECT course_id, AVG(video_duration_sec) AS avg_duration
      FROM topic_stats
      WHERE snapshot_date = (SELECT MAX(snapshot_date) FROM topic_stats)
      GROUP BY course_id
    ) ts_avg ON ts_avg.course_id = cs.course_id
    WHERE cs.snapshot_date = (SELECT MAX(snapshot_date) FROM course_snapshots)
      AND cs.instructor_id IS NOT NULL
    ON CONFLICT (instructor_id, course_id, snapshot_date) DO UPDATE SET
      feedback_pct = EXCLUDED.feedback_pct,
      avg_video_length_sec = EXCLUDED.avg_video_length_sec,
      completion_rate = EXCLUDED.completion_rate
  `);

  // Compute trend_7d separately (compare today vs 7 days ago)
  await pool.query(`
    UPDATE instructor_scores curr SET
      trend_7d = curr.feedback_pct - prev.feedback_pct
    FROM instructor_scores prev
    WHERE prev.instructor_id = curr.instructor_id
      AND prev.course_id = curr.course_id
      AND prev.snapshot_date = CURRENT_DATE - 7
      AND curr.snapshot_date = CURRENT_DATE
      AND curr.feedback_pct IS NOT NULL
      AND prev.feedback_pct IS NOT NULL
  `);
}

async function computeCohortStats(pool) {
  await pool.query(`
    UPDATE cohorts c SET
      user_count = sub.cnt,
      avg_retention_30d = sub.retention,
      churn_rate = sub.churn,
      updated_at = NOW()
    FROM (
      SELECT
        cm.cohort_id,
        COUNT(DISTINCT cm.user_id) AS cnt,
        AVG(CASE WHEN da.activity_days >= 1 THEN 1.0 ELSE 0.0 END) * 100 AS retention,
        AVG(CASE WHEN da.activity_days = 0 THEN 1.0 ELSE 0.0 END) * 100 AS churn
      FROM cohort_members cm
      LEFT JOIN (
        SELECT user_id, COUNT(DISTINCT date) AS activity_days
        FROM daily_activity
        WHERE date >= CURRENT_DATE - INTERVAL '30 days'
        GROUP BY user_id
      ) da ON da.user_id = cm.user_id
      GROUP BY cm.cohort_id
    ) sub
    WHERE c.id = sub.cohort_id
  `);
}

async function computeSessionPatterns(pool) {
  await pool.query(`
    INSERT INTO session_patterns (user_id, devices_7d, device_switches_7d, ip_count_7d, anomaly_score, snapshot_date)
    SELECT
      e.user_id,
      COUNT(DISTINCT (e.payload->>'device_id')) AS devices_7d,
      COUNT(DISTINCT (e.payload->>'device_id')) - 1 AS device_switches_7d,
      COUNT(DISTINCT (e.payload->>'ip')) AS ip_count_7d,
      CASE
        WHEN COUNT(DISTINCT (e.payload->>'device_id')) > 3 THEN 80
        WHEN COUNT(DISTINCT (e.payload->>'ip')) > 5 THEN 60
        ELSE 0
      END AS anomaly_score,
      CURRENT_DATE AS snapshot_date
    FROM events e
    WHERE e.event_type IN ('device.registered', 'device.switched', 'session.started')
      AND e.created_at >= CURRENT_DATE - INTERVAL '7 days'
      AND e.user_id IS NOT NULL
    GROUP BY e.user_id
    ON CONFLICT (user_id, snapshot_date) DO UPDATE SET
      devices_7d = EXCLUDED.devices_7d,
      device_switches_7d = EXCLUDED.device_switches_7d,
      ip_count_7d = EXCLUDED.ip_count_7d,
      anomaly_score = EXCLUDED.anomaly_score
  `);
}

async function computeSummary(pool) {
  await pool.query(`
    INSERT INTO summary (date, action_count, watch_count, win_count, computed_at)
    SELECT
      CURRENT_DATE,
      COUNT(*) FILTER (WHERE severity = 'action'),
      COUNT(*) FILTER (WHERE severity = 'watch'),
      COUNT(*) FILTER (WHERE severity = 'win'),
      NOW()
    FROM insights
    WHERE NOT is_dismissed
      AND (expires_at IS NULL OR expires_at > NOW())
    ON CONFLICT (date) DO UPDATE SET
      action_count = EXCLUDED.action_count,
      watch_count = EXCLUDED.watch_count,
      win_count = EXCLUDED.win_count,
      computed_at = NOW()
  `);
}

async function refreshPulseDeltas(pool) {
  // Compute 7-day deltas for key pulse metrics by comparing today vs 7 days ago
  // Revenue delta: compare this week's revenue to last week's
  await pool.query(`
    WITH weekly AS (
      SELECT
        SUM(amount) FILTER (WHERE created_at >= CURRENT_DATE - 7) AS this_week,
        SUM(amount) FILTER (WHERE created_at >= CURRENT_DATE - 14 AND created_at < CURRENT_DATE - 7) AS last_week
      FROM orders WHERE status = 'paid'
    )
    UPDATE pulse SET
      delta = CASE
        WHEN w.last_week > 0 THEN ROUND(((w.this_week - w.last_week) / w.last_week * 100))::text || '%'
        ELSE NULL
      END,
      updated_at = NOW()
    FROM weekly w
    WHERE section = 'revenue' AND key = 'revenue_today'
  `);

  // Active users delta: compare this week's DAU to last week's
  await pool.query(`
    WITH weekly AS (
      SELECT
        COUNT(DISTINCT user_id) FILTER (WHERE date >= CURRENT_DATE - 7) AS this_week,
        COUNT(DISTINCT user_id) FILTER (WHERE date >= CURRENT_DATE - 14 AND date < CURRENT_DATE - 7) AS last_week
      FROM daily_activity
    )
    UPDATE pulse SET
      delta = CASE
        WHEN w.last_week > 0 THEN ROUND(((w.this_week::decimal - w.last_week) / w.last_week * 100))::text || '%'
        ELSE NULL
      END,
      updated_at = NOW()
    FROM weekly w
    WHERE section = 'retention' AND key = 'active_users_7d'
  `);

  // Record pipeline last run
  await pool.query(`
    INSERT INTO pulse (section, key, value, label, updated_at)
    VALUES ('system', 'pipeline_last_run', EXTRACT(EPOCH FROM NOW())::numeric, 'Pipeline Last Run', NOW())
    ON CONFLICT (section, key) DO UPDATE SET
      value = EXCLUDED.value,
      updated_at = NOW()
  `);
}

const steps = [
  { name: 'streak_history', fn: computeStreaks },
  { name: 'user_profiles_streaks', fn: updateUserStreaks },
  { name: 'instructor_scores', fn: computeInstructorScores },
  { name: 'cohort_stats', fn: computeCohortStats },
  { name: 'session_patterns', fn: computeSessionPatterns },
  { name: 'summary', fn: computeSummary },
  { name: 'pulse_deltas', fn: refreshPulseDeltas },
];

// Main
if (process.argv[1]?.endsWith('compute.js')) {
  runSteps(db, steps, reportHealth)
    .then(() => {
      console.log('Compute: all steps completed.');
      return db.end();
    })
    .catch(async (err) => {
      console.error('Compute: pipeline failed:', err.message);
      await db.end();
      process.exit(1);
    });
}
```

- [ ] **Step 4: Run tests**

```bash
npx vitest run scripts/__tests__/compute.test.js
```

Expected: 2 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/compute.js scripts/__tests__/compute.test.js
git commit -m "feat: add compute script (7 nightly computation steps)"
```

---

### Task 19: Cleanup Script

**Files:**
- Create: `qimah-eyes/scripts/cleanup.js`

- [ ] **Step 1: Write cleanup.js**

```js
// scripts/cleanup.js
import db from './lib/db.js';
import { reportHealth } from './lib/health.js';

async function main() {
  const start = Date.now();
  try {
    // Delete old events (90-day retention)
    const eventsResult = await db.query(
      `DELETE FROM events WHERE created_at < NOW() - INTERVAL '90 days'`
    );
    console.log(`Cleanup: deleted ${eventsResult.rowCount} old events`);

    // Delete resolved dead letters (30-day retention)
    const dlResult = await db.query(
      `DELETE FROM dead_letters WHERE resolved AND created_at < NOW() - INTERVAL '30 days'`
    );
    console.log(`Cleanup: deleted ${dlResult.rowCount} resolved dead letters`);

    // VACUUM ANALYZE
    await db.query('VACUUM ANALYZE events');
    await db.query('VACUUM ANALYZE dead_letters');
    console.log('Cleanup: VACUUM ANALYZE complete');

    await reportHealth(db, 'cleanup', 'ok', Date.now() - start);
  } catch (err) {
    await reportHealth(db, 'cleanup', 'failed', Date.now() - start, err.message);
    console.error('Cleanup failed:', err.message);
    process.exit(1);
  } finally {
    await db.end();
  }
}

main();
```

- [ ] **Step 2: Commit**

```bash
git add scripts/cleanup.js
git commit -m "feat: add cleanup script (event retention + VACUUM)"
```

---

## Chunk 5: Intelligence, Backfill & Orchestration

### Task 20: Insight Rules (Module Structure)

**Files:**
- Create: `qimah-eyes/scripts/rules/revenue.js`
- Create: `qimah-eyes/scripts/rules/learning.js`
- Create: `qimah-eyes/scripts/rules/retention.js`
- Create: `qimah-eyes/scripts/rules/security.js`
- Create: `qimah-eyes/scripts/rules/community.js`
- Create: `qimah-eyes/scripts/rules/operations.js`
- Create: `qimah-eyes/scripts/rules/recruitment.js`
- Create: `qimah-eyes/scripts/rules/comms.js`
Each rule module exports an array of rule objects with `{ id, section, evaluate(db) }`. The evaluate function returns an array of insight objects. Marketing-related insights (funnel, conversion) are covered under revenue and retention rules.

- [ ] **Step 1: Write revenue rules (highest priority)**

```js
// scripts/rules/revenue.js
export default [
  {
    id: 'revenue.payment_failures_spike',
    section: 'revenue',
    async evaluate(db) {
      const { rows } = await db.query(`
        SELECT
          COUNT(*) FILTER (WHERE created_at >= CURRENT_DATE - INTERVAL '7 days') AS recent,
          COUNT(*) FILTER (WHERE created_at >= CURRENT_DATE - INTERVAL '28 days') / 4.0 AS avg_weekly
        FROM payment_failures
      `);
      const r = rows[0];
      if (r.avg_weekly > 0 && r.recent > r.avg_weekly * 1.5) {
        return [{
          severity: 'watch',
          headline: `Payment failures spiked: ${r.recent} this week vs ${Math.round(r.avg_weekly)} avg`,
          body: `Payment failures are ${Math.round((r.recent / r.avg_weekly - 1) * 100)}% above the 4-week average.`,
          evidence: 'Source: Moyasar payment_failures',
          actions: [{ label: 'View failures', url: '/revenue', type: 'primary' }],
        }];
      }
      return [];
    }
  },
  {
    id: 'revenue.revenue_anomaly',
    section: 'revenue',
    async evaluate(db) {
      const { rows } = await db.query(`
        WITH weekly AS (
          SELECT
            SUM(amount) FILTER (WHERE created_at >= CURRENT_DATE - INTERVAL '7 days') AS this_week,
            SUM(amount) FILTER (WHERE created_at >= CURRENT_DATE - INTERVAL '14 days'
                                  AND created_at < CURRENT_DATE - INTERVAL '7 days') AS last_week
          FROM orders WHERE status = 'paid'
        )
        SELECT * FROM weekly WHERE last_week > 0
      `);
      if (rows.length === 0) return [];
      const r = rows[0];
      const pctChange = ((r.this_week - r.last_week) / r.last_week) * 100;
      if (Math.abs(pctChange) > 25) {
        return [{
          severity: 'watch',
          headline: `Revenue ${pctChange > 0 ? 'up' : 'down'} ${Math.abs(Math.round(pctChange))}% week-over-week`,
          body: `This week: ${r.this_week} SAR vs last week: ${r.last_week} SAR.`,
          evidence: 'Source: orders table',
          actions: [{ label: 'View revenue', url: '/revenue', type: 'primary' }],
        }];
      }
      return [];
    }
  },
];
```

- [ ] **Step 2: Write remaining rule modules (skeleton + highest-value rules)**

Implement at least the top 10 rules per parent spec Section 6. Remaining rules can be stubbed as empty arrays and filled in later. Priority rules:

1. `revenue.payment_failures_spike` (done above)
2. `revenue.revenue_anomaly` (done above)
3. `learning.instructor_feedback_drop`
4. `learning.negative_feedback_cluster`
5. `retention.inactive_students`
6. `retention.at_risk_detection`
7. `security.sharing_alert`
8. `ops.cron_failure`
9. `recruitment.sla_breach`
10. `comms.open_rate_anomaly`

Each rule module follows the same pattern. See parent spec Section 6 for SQL queries and trigger conditions.

- [ ] **Step 3: Commit**

```bash
git add scripts/rules/
git commit -m "feat: add insight rules (10 priority rules + module structure)"
```

---

### Task 21: Insights Script

**Files:**
- Create: `qimah-eyes/scripts/insights.js`

- [ ] **Step 1: Write insights.js**

```js
// scripts/insights.js
import db from './lib/db.js';
import { reportHealth } from './lib/health.js';

// Import all rule modules
import revenueRules from './rules/revenue.js';
import learningRules from './rules/learning.js';
import retentionRules from './rules/retention.js';
import securityRules from './rules/security.js';
import communityRules from './rules/community.js';
import operationsRules from './rules/operations.js';
import recruitmentRules from './rules/recruitment.js';
import commsRules from './rules/comms.js';
const allRules = [
  ...revenueRules, ...learningRules, ...retentionRules,
  ...securityRules, ...communityRules, ...operationsRules,
  ...recruitmentRules, ...commsRules,
];

async function main() {
  const start = Date.now();
  let totalInsights = 0;

  try {
    for (const rule of allRules) {
      try {
        const insights = await rule.evaluate(db);
        for (const insight of insights) {
          // Dedup: check if active insight with same rule_id exists
          const { rows: existing } = await db.query(
            `SELECT id FROM insights
             WHERE rule_id = $1 AND NOT is_dismissed
               AND (expires_at IS NULL OR expires_at > NOW())
             LIMIT 1`,
            [rule.id]
          );

          if (existing.length > 0) {
            // Update existing insight with fresh data
            await db.query(
              `UPDATE insights SET
                body = $1, evidence = $2, actions = $3::jsonb, updated_at = NOW()
               WHERE id = $4`,
              [insight.body, insight.evidence, JSON.stringify(insight.actions), existing[0].id]
            );
          } else {
            // Insert new insight
            await db.query(
              `INSERT INTO insights (section, severity, headline, body, evidence, actions, rule_id, expires_at, created_at)
               VALUES ($1, $2, $3, $4, $5, $6::jsonb, $7, NOW() + INTERVAL '7 days', NOW())`,
              [rule.section, insight.severity, insight.headline, insight.body,
               insight.evidence, JSON.stringify(insight.actions), rule.id]
            );
          }
          totalInsights++;
        }
      } catch (err) {
        console.error(`Rule ${rule.id} failed:`, err.message);
        // Continue with other rules - don't let one failure stop everything
      }
    }

    console.log(`Insights: evaluated ${allRules.length} rules, produced ${totalInsights} insights`);
    await reportHealth(db, 'insights', 'ok', Date.now() - start);
  } catch (err) {
    await reportHealth(db, 'insights', 'failed', Date.now() - start, err.message);
    process.exit(1);
  } finally {
    await db.end();
  }
}

main();
```

- [ ] **Step 2: Commit**

```bash
git add scripts/insights.js
git commit -m "feat: add insights engine (rule evaluation + dedup)"
```

---

### Task 22: AI Weekly Script

**Files:**
- Create: `qimah-eyes/scripts/ai-weekly.js`

- [ ] **Step 1: Write ai-weekly.js**

```js
// scripts/ai-weekly.js
import db from './lib/db.js';
import { reportHealth } from './lib/health.js';
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

async function collectWeeklyData(pool) {
  const [revenue, learning, retention, activeInsights] = await Promise.all([
    pool.query(`
      SELECT SUM(amount) AS total, COUNT(*) AS orders,
             COUNT(*) FILTER (WHERE status = 'refunded') AS refunds
      FROM orders WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
    `),
    pool.query(`
      SELECT instructor_name, course_id, feedback_pct, trend_7d, completion_rate
      FROM instructor_scores WHERE snapshot_date = (SELECT MAX(snapshot_date) FROM instructor_scores)
    `),
    pool.query(`
      SELECT
        COUNT(DISTINCT user_id) FILTER (WHERE date >= CURRENT_DATE - 1) AS dau,
        COUNT(DISTINCT user_id) FILTER (WHERE date >= CURRENT_DATE - 7) AS wau,
        COUNT(DISTINCT user_id) FILTER (WHERE date >= CURRENT_DATE - 30) AS mau
      FROM daily_activity
    `),
    pool.query(`
      SELECT section, severity, headline FROM insights
      WHERE NOT is_dismissed AND created_at >= CURRENT_DATE - INTERVAL '7 days'
      ORDER BY created_at DESC LIMIT 20
    `),
  ]);

  return {
    revenue: revenue.rows[0],
    instructors: learning.rows,
    retention: retention.rows[0],
    activeInsights: activeInsights.rows,
  };
}

async function main() {
  const start = Date.now();
  try {
    const data = await collectWeeklyData(db);

    const response = await client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 2048,
      system: `You are an analytics advisor for Qimah, an online education platform in Saudi Arabia.`,
      messages: [{
        role: 'user',
        content: `Here is this week's data summary:\n${JSON.stringify(data, null, 2)}\n\nFor each section (Revenue, Learning, Retention, Security, Community, Operations, Recruitment, Comms), produce 0-2 insights that identify patterns the rules missed. Also produce 1-2 cross-section insights. Output as JSON array of {section, headline, body} objects.`
      }],
    });

    const text = response.content[0].text;
    const jsonMatch = text.match(/\[[\s\S]*\]/);
    if (!jsonMatch) throw new Error('No JSON array in AI response');

    const insights = JSON.parse(jsonMatch[0]);
    for (const insight of insights) {
      await db.query(
        `INSERT INTO insights (section, severity, headline, body, is_ai, expires_at, created_at)
         VALUES ($1, 'info', $2, $3, true, NOW() + INTERVAL '7 days', NOW())`,
        [insight.section, insight.headline, insight.body]
      );
    }

    console.log(`AI Weekly: generated ${insights.length} insights`);
    await reportHealth(db, 'ai_weekly', 'ok', Date.now() - start);
  } catch (err) {
    await reportHealth(db, 'ai_weekly', 'failed', Date.now() - start, err.message);
    console.error('AI Weekly failed:', err.message);
    process.exit(1);
  } finally {
    await db.end();
  }
}

main();
```

- [ ] **Step 2: Install Anthropic SDK**

```bash
npm install @anthropic-ai/sdk
```

- [ ] **Step 3: Commit**

```bash
git add scripts/ai-weekly.js package.json package-lock.json
git commit -m "feat: add weekly AI analysis script (Claude Sonnet)"
```

---

### Task 23: Backfill Script

**Files:**
- Create: `qimah-eyes/scripts/backfill.js`

- [ ] **Step 1: Write backfill.js**

```js
// scripts/backfill.js
import db from './lib/db.js';

const TARGETS = {
  orders: backfillOrders,
  customers: backfillCustomers,
  coupons: backfillCoupons,
  user_profiles: backfillUserProfiles,
  course_snapshots: backfillCourseSnapshots,
  conversions: backfillConversions,
  email: backfillEmail,
  clickup: backfillClickUp,
  video_pipeline: backfillVideoPipeline,
};

async function backfillOrders(pool, dryRun) {
  // Paginate through WooCommerce REST API
  const baseUrl = process.env.WP_BASE_URL.replace('/wp-json', '');
  const auth = Buffer.from(`${process.env.WC_CONSUMER_KEY}:${process.env.WC_CONSUMER_SECRET}`).toString('base64');
  let page = 1;
  let total = 0;

  while (true) {
    const res = await fetch(
      `${baseUrl}/wp-json/wc/v3/orders?per_page=100&page=${page}&status=completed,refunded`,
      { headers: { Authorization: `Basic ${auth}` } }
    );
    const orders = await res.json();
    if (!orders.length) break;

    for (const order of orders) {
      if (dryRun) {
        console.log(`[DRY RUN] Would upsert order ${order.id}`);
      } else {
        await pool.query(
          `INSERT INTO orders (wp_order_id, user_id, amount, currency, payment_method, status, coupon_code, created_at, updated_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
           ON CONFLICT (wp_order_id) DO UPDATE SET
             status = EXCLUDED.status, amount = EXCLUDED.amount, updated_at = NOW()`,
          [order.id, order.customer_id, order.total, order.currency,
           order.payment_method, order.status === 'completed' ? 'paid' : 'refunded',
           order.coupon_lines?.[0]?.code || null, order.date_created]
        );

        // Insert order items
        for (const item of order.line_items) {
          await pool.query(
            `INSERT INTO order_items (order_id, course_id, course_title, price, discount_amount)
             VALUES ((SELECT id FROM orders WHERE wp_order_id = $1), $2, $3, $4, $5)
             ON CONFLICT DO NOTHING`,
            [order.id, item.product_id, item.name, item.total, item.subtotal - item.total]
          );
        }
      }
      total++;
    }

    console.log(`Orders: page ${page}, processed ${total} so far`);
    page++;
    await new Promise(r => setTimeout(r, 1000)); // Rate limit: 1 req/sec
  }

  console.log(`Orders: backfilled ${total} orders`);
}

// Stub other backfill functions - same pattern: paginate API, upsert
async function backfillCustomers(pool, dryRun) {
  console.log('Customers: derived from orders (run orders first, then compute)');
  if (dryRun) return;
  await pool.query(`
    INSERT INTO customers (user_id, email, first_order_at, total_spent, order_count, segment, created_at, updated_at)
    SELECT
      user_id,
      NULL,
      MIN(created_at),
      SUM(CASE WHEN status = 'paid' THEN amount ELSE -amount END),
      COUNT(*),
      CASE WHEN COUNT(*) >= 3 THEN 'vip' WHEN COUNT(*) >= 2 THEN 'returning' ELSE 'new' END,
      MIN(created_at),
      NOW()
    FROM orders
    GROUP BY user_id
    ON CONFLICT (user_id) DO UPDATE SET
      total_spent = EXCLUDED.total_spent,
      order_count = EXCLUDED.order_count,
      segment = EXCLUDED.segment,
      updated_at = NOW()
  `);
}

async function backfillCoupons(pool, dryRun) { console.log('TODO: implement coupons backfill'); }
async function backfillUserProfiles(pool, dryRun) { console.log('TODO: implement user_profiles backfill'); }
async function backfillCourseSnapshots(pool, dryRun) { console.log('TODO: implement course_snapshots backfill'); }
async function backfillConversions(pool, dryRun) { console.log('TODO: implement conversions backfill'); }
async function backfillEmail(pool, dryRun) { console.log('TODO: implement email backfill'); }
async function backfillClickUp(pool, dryRun) { console.log('TODO: implement ClickUp backfill'); }
async function backfillVideoPipeline(pool, dryRun) { console.log('TODO: implement video_pipeline backfill'); }

// CLI
const args = process.argv.slice(2);
const targetArg = args.find(a => a.startsWith('--target='));
const dryRun = args.includes('--dry-run');
const confirm = args.includes('--confirm');

if (!targetArg) {
  console.log('Usage: node scripts/backfill.js --target=<table> [--dry-run|--confirm]');
  console.log('Available targets:', Object.keys(TARGETS).join(', '));
  process.exit(0);
}

const target = targetArg.split('=')[1];
if (!TARGETS[target]) {
  console.error(`Unknown target: ${target}`);
  process.exit(1);
}

if (!dryRun && !confirm) {
  console.error('Must specify --dry-run or --confirm');
  process.exit(1);
}

console.log(`Backfill: ${target} (${dryRun ? 'DRY RUN' : 'LIVE'})`);
TARGETS[target](db, dryRun)
  .then(() => db.end())
  .catch(async (err) => {
    console.error('Backfill failed:', err.message);
    await db.end();
    process.exit(1);
  });
```

- [ ] **Step 2: Commit**

```bash
git add scripts/backfill.js
git commit -m "feat: add backfill script (orders complete, others stubbed)"
```

---

### Task 24: Orchestrator n8n Workflow

**Files:**
- Create: `qimah-eyes/n8n/workflows/orchestrator-nightly.json`

The orchestrator workflow ties everything together. It triggers the nightly computation pipeline after the gate passes.

**Workflow nodes:**

```
Schedule (00:50 UTC daily)
  -> Execute Command: node scripts/gate.js
     -> On success (exit 0):
        -> Execute Command: node scripts/compute.js
        -> Execute Command: node scripts/insights.js
        -> Execute Command: node scripts/cleanup.js
     -> On failure (exit 1):
        -> Send notification (email via Resend or n8n notification)

Schedule (Sunday 02:00 UTC)
  -> Execute Command: node scripts/ai-weekly.js
```

- [ ] **Step 1: Create orchestrator workflow in n8n UI**

Add Schedule trigger at 00:50 UTC. Chain Execute Command nodes for gate -> compute -> insights -> cleanup. Add error notification.

- [ ] **Step 2: Add Sunday-only AI weekly trigger**

Separate Schedule trigger for Sunday 02:00 UTC, triggering `node scripts/ai-weekly.js`.

- [ ] **Step 3: Test the orchestration chain manually**

Run each script individually first, then test the full chain.

- [ ] **Step 4: Export and commit**

```bash
git add n8n/workflows/orchestrator-nightly.json
git commit -m "feat: add nightly orchestrator workflow"
```

---

### Task 25: Credentials Example File

**Files:**
- Create: `qimah-eyes/n8n/credentials.example.json`

- [ ] **Step 1: Create credentials example**

```json
{
  "note": "This file documents which n8n credentials are needed. Create them in the n8n UI.",
  "credentials": [
    {
      "name": "Qimah WP API",
      "type": "httpHeaderAuth",
      "headerName": "X-Qimah-API-Key",
      "description": "Used by WP Learning and WP Users pull workflows"
    },
    {
      "name": "Moyasar API",
      "type": "httpHeaderAuth",
      "headerName": "Authorization",
      "description": "Bearer token for Moyasar payments API"
    },
    {
      "name": "Bunny CDN API",
      "type": "httpHeaderAuth",
      "headerName": "AccessKey",
      "description": "Bunny Stream API key"
    },
    {
      "name": "Resend API",
      "type": "httpHeaderAuth",
      "headerName": "Authorization",
      "description": "Bearer token for Resend email API"
    },
    {
      "name": "ClickUp API",
      "type": "httpHeaderAuth",
      "headerName": "Authorization",
      "description": "ClickUp API token"
    },
    {
      "name": "PostHog API",
      "type": "httpHeaderAuth",
      "headerName": "Authorization",
      "description": "PostHog personal API key"
    },
    {
      "name": "Qimah Eyes Postgres",
      "type": "postgres",
      "description": "Connection to Qimah Eyes Postgres database"
    }
  ]
}
```

- [ ] **Step 2: Commit**

```bash
git add n8n/credentials.example.json
git commit -m "docs: add credentials example for n8n setup"
```

---

### Task 26: Final Integration Test

- [ ] **Step 1: Deploy schema to VPS Postgres**

```bash
psql -U qimah_eyes -d qimah_eyes -f sql/schema.sql
```

- [ ] **Step 2: Test webhook workflow end-to-end**

Send a sample `commerce.order_paid` webhook to the Revenue workflow URL. Verify data appears in `orders`, `customers`, `events`, `pulse`, and `cron_health` tables.

- [ ] **Step 3: Test API pull workflow**

Manually trigger one API pull workflow (e.g., WP Users). Verify `user_profiles` and `cron_health` are populated.

- [ ] **Step 4: Test gate -> compute -> insights chain**

Seed `cron_health` with `ok` status for all 7 API pull jobs, then run:

```bash
node scripts/gate.js && node scripts/compute.js && node scripts/insights.js
```

Verify `streak_history`, `instructor_scores`, `session_patterns`, `summary`, and `insights` tables have data.

- [ ] **Step 5: Test backfill (dry run)**

```bash
node scripts/backfill.js --target=orders --dry-run
```

Verify output shows orders that would be imported.

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "chore: integration test verification complete"
```
