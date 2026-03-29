# GOD Dashboard - Design Spec

**Date:** 2026-03-13
**Status:** Draft (rev 5 - spec review approved)
**Author:** Claude (brainstormed with M)

---

## 1. Summary

A centralized analytics and decision-support dashboard for Qimah, running on a dedicated VPS. Nine section tabs, each showing actionable insights (not vanity metrics) powered by rule-based daily analysis and weekly LLM-generated cross-section intelligence. Designed for daily and weekly team meetings - open, scan what needs attention, act on it.

**This is not a metrics dashboard. It's a decision engine.**

---

## 2. Infrastructure

### VPS: Hetzner CPX41

| Service | Role | RAM |
|---------|------|-----|
| PostHog (self-hosted) | Pre-login: funnels, session replays, autocapture | 5-6 GB |
| n8n | Ingestion: receives webhooks, pulls APIs, writes to Postgres | 512 MB |
| Postgres 16 | Analytics DB: raw data + computed insights | 1.5 GB |
| Next.js Dashboard | GOD dashboard UI (9 sections, RTL Arabic) | 256 MB |
| FMS | Find My Schedule (already exists) | 256 MB |
| Insight Engine | Nightly cron: rule evaluation + weekly Claude API | shared |
| OS + headroom | | 2 GB |

**Total: ~10.5 GB of 16 GB** - comfortable headroom.

### Data Flow

```
PRE-LOGIN (anonymous)                POST-LOGIN (authenticated)
━━━━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━━━━━━━━━━━━━━━
PostHog JS snippet                   WP webhooks (50+ event types)
    │                                    │
    ▼                                    ▼
PostHog (ClickHouse)                 n8n (webhook receiver)
    │                                    │
    │  nightly sync                      │  real-time write
    ▼                                    ▼
n8n ──────────────────────────────► Postgres
                                        ▲
External APIs (pulled by n8n):          │
  • Bunny CDN (video stats) ────────────┤
  • Resend (email metrics) ─────────────┤
  • ClickUp (recruitment) ──────────────┤
  • WP REST API (snapshots) ────────────┘
                                        │
                                        ▼
                                  Insight Engine
                                  (nightly rules + weekly LLM)
                                        │
                                        ▼
                                  insights table
                                        │
                                        ▼
                                  Next.js Dashboard
```

**Key principle:** WordPress is a dumb data source. All intelligence lives on the VPS. The dashboard never queries WP directly.

### WP-Side Requirements

From the existing analytics API spec (`2026-03-11-analytics-api-design.md`):
- 2 new REST endpoints: `GET /analytics/enrollments`, `GET /analytics/feedback`
- New `read:analytics` API scope
- New `user.registered` webhook event
- **Phase 1 (critical):** Wire `commerce.order_paid`, `commerce.order_refunded`, `session.started` - these are prerequisites for real-time Pulse and Revenue sections
- **Phase 4:** Wire remaining dead webhook events (~15 events - bookings, circles, subscriptions, etc. See analytics API spec for full list)

No other WP changes needed. Everything else flows through existing webhooks or VPS-side API pulls.

### Timezone Policy

All timestamps stored as `TIMESTAMP` (UTC assumed) in Postgres. A few columns use `TIMESTAMPTZ` explicitly where source data arrives with timezone info; in practice, both behave identically since n8n normalizes to UTC before writing. Sources:
- WP webhooks: +03:00 (AST) - converted to UTC on ingestion
- PostHog: UTC (native)
- Bunny/Resend/ClickUp: UTC (native)

Display: converted to AST (+03:00) in the Next.js frontend. Nightly cron runs at 03:00 AST (00:00 UTC).

### Postgres Backup

Daily `pg_dump` to Hetzner Object Storage (S3-compatible) via cron at 05:00 UTC. 30-day retention. Tested restore monthly.

---

## 3. Dashboard UI

### Structure

```
┌─────────────────────────────────────────────────────┐
│  Qimah GOD Dashboard              March 13, 2026    │
├─────────────────────────────────────────────────────┤
│  3 need action  │  4 watching  │  5 wins            │  ← summary bar
├─────────────────────────────────────────────────────┤
│ Revenue● │ Learning● │ Retention● │ Pulse │ ...     │  ← tabs with alert dots
├─────────────────────────────────────────────────────┤
│                                                     │
│  [Pulse strip: 4 live numbers]                      │
│                                                     │
│  [NEEDS ACTION insight cards]                       │
│  [WATCH insight cards]                              │
│  [WIN insight cards]                                │
│  [INFO insight cards]                               │
│                                                     │
│  [Weekly AI Analysis block]                         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### UX Principles

- **Tabs have alert dots** - red dot = has NEEDS ACTION items, yellow = has WATCH items, no dot = clean
- **Summary bar** at top shows cross-section totals so you know the day's weight before clicking anything
- **Each section leads with insights, not charts** - numbers support the insight, not the other way around
- **Pulse strip** at top of each section shows 4 key live numbers for context
- **Action buttons** on insight cards link to the thing you'd do about it (send message, open pipeline, view feedback)
- **Weekly AI Analysis** block at bottom of each section with Claude-generated cross-domain insight
- **RTL Arabic** throughout, with English for technical terms (same style as qimah.net)
- **Admin + team view** for now. Instructor scoping is V2 (data model supports it via `instructor_id` columns)

### Insight Card Anatomy

```
┌──────────────────────────────────────────┐
│  NEEDS ACTION                            │  ← severity badge
│                                          │
│  Instructor Khalid: feedback dropped     │  ← headline (bold)
│  89% → 71% after Week 6                 │
│                                          │
│  8 negative comments mention "unclear    │  ← body (detail + context)
│  explanations". Correlates with format   │
│  change: 45min → 20min, no slides.      │
│                                          │
│  Source: CEP feedback + Bunny metadata   │  ← evidence line
│                                          │
│  [View feedback]  [Message Khalid]       │  ← action buttons
└──────────────────────────────────────────┘
```

### Severity Levels

| Level | Meaning | Expiry |
|-------|---------|--------|
| `action` | Requires a decision or response this week | 7 days |
| `watch` | Trending in a concerning direction, monitor | 14 days |
| `win` | Something went right, worth acknowledging | 7 days |
| `info` | Pattern or insight, no action needed | 14 days |

---

## 4. The Nine Sections

### 4.1 Revenue

**Live pulse:** Revenue today, Orders this week, Avg order value, Refunds

**Insight types (rule-based):**
- Discount code effectiveness by customer segment (new vs returning)
- Bundle opportunities: course co-purchase overlap above threshold
- Payment method failure spikes
- Revenue trend anomalies (week-over-week, vs same period last year)
- Coupon ROI: revenue generated vs discount given
- ROAS: ad spend vs revenue from campaign-attributed students

**Weekly AI analysis examples:**
- "Revenue dip is seasonal - same week last year was -28%. But AOV is climbing."
- "15% codes work for returning, 10% for new. Segment your next campaign."

**Data needed:** orders, order_items, coupons, payment_failures, customers, conversions (PostHog bridge)

### 4.2 Learning

**Live pulse:** Avg feedback %, Topics completed (7d), Avg watch time, Active courses

**Insight types:**
- Instructor feedback trend alerts (drop > 10 points triggers action)
- Video length sweet spot analysis (completion rate by duration bucket)
- Course completion rate changes after content updates
- Topics with highest negative feedback (specific complaints surfaced)
- Per-instructor comparison: feedback, completion rate, video length

**Weekly AI analysis examples:**
- "Khalid's drop correlates with video length change, not content quality - quiz scores stayed the same."
- "Videos 25-35 min get 23% higher completion. Under 15 min: students skip ahead."

**Data sources:**
- `course_snapshots`, `topic_stats`, `instructor_scores` - nightly API pull
- `feedback` - webhook (`feedback.submitted`) + nightly API snapshot
- `video_events` - **PostHog autocapture on Bunny iframe** (not WP-side). PostHog's `$autocapture` tracks play/pause/seek events on the embedded player. Nightly PostHog API sync extracts per-user video engagement (watch duration, % watched) and writes to `video_events`. This avoids any WP plugin changes. If PostHog autocapture proves insufficient for iframe events, V2 can add explicit CEP-side tracking via `player.on('timeupdate')` → PostHog custom event.

### 4.3 Retention

**Live pulse:** NZD yesterday %, DAU, WAU, WAU/MAU ratio

**Insight types:**
- Inactive student alerts: active students who stopped logging in (configurable threshold: 7 days default)
- Cohort churn rates: campaign vs organic vs batch
- Campaign-attributed student retention vs organic
- Streak milestone distribution
- At-risk student identification (declining activity pattern before full dropout)

**Weekly AI analysis examples:**
- "Paid campaign students churn 2.3x more after week 4. Consider onboarding drip sequence."
- "Batch 7 dropout cluster: 10/12 had no prior Qimah contact."

**Data needed:** user_profiles, daily_activity, cohorts, cohort_members, streak_history, conversions

### 4.4 Pulse (Real-Time)

**Live pulse:** Online now, Revenue today, Active course breakdown, Pending applicants

**No insight cards** - this section is pure live metrics. It's the "glance" section. Near-real-time updates via n8n writing to the pulse table on every relevant webhook.

**Data needed:** pulse table (key-value, updated on each webhook)

### 4.5 Security

**Live pulse:** Sharing alerts, Watchlist count, Active sessions (7d), Device switches (7d)

**Insight types:**
- New sharing detection alerts
- Unusual device switch patterns
- IP anomaly clusters
- Watchlist status changes

**Data needed:** sharing_alerts, session_patterns

### 4.6 Community

**Live pulse:** Active circles, Circle members, Challenge completion %

**Insight types:**
- Challenge format effectiveness (shared_streak vs total_topics completion rates)
- Circle engagement trends

**Discord activity: deferred to V2.** The existing Discord integration (qimah-profile) handles OAuth only - no activity data (messages, voice minutes). Tracking Discord activity requires deploying a bot to the Qimah Discord server, which is out of scope for V1. The `discord_activity` table is included in the schema for forward-compatibility but will remain empty until a bot is deployed.

**Data needed:** circles, circle_activity (discord_activity in V2)

### 4.7 Operations

**Live pulse:** Video pipeline status, Failed uploads, Videos processed (7d), Cron health

**Insight types:**
- Video processing failures or slowdowns
- Cron job failures
- Storage/bandwidth trends

**Data needed:** video_pipeline, cron_health

### 4.8 Recruitment

**Live pulse:** Pending review, In pipeline, Hired this round, Ghosted

**Insight types:**
- Applicants exceeding review SLA (default: 3 days)
- Ghost rate by track
- Prior Discord activity as predictor of hire retention
- Pipeline bottleneck identification (which stage has the most stuck applicants)

**Weekly AI analysis examples:**
- "Applicants with existing Discord activity have 4x better retention after hiring."

**Data needed:** applicants, pipeline_snapshots

### 4.9 Comms

**Live pulse:** Email delivery rate, Open rate (7d), Emails sent (7d), Bounces

**Insight types:**
- Open rate anomalies (significant deviation from baseline)
- Best send time analysis
- Subject line performance comparison
- Segment engagement differences

**Data needed:** email_campaigns, email_events

---

## 5. Postgres Schema

### 5.1 PostHog Bridge (pre-login → post-login)

```sql
-- Anonymous visitors (synced from PostHog nightly via n8n)
CREATE TABLE visitors (
  id              SERIAL PRIMARY KEY,
  anonymous_id    VARCHAR(100) UNIQUE NOT NULL,  -- PostHog distinct_id
  first_seen      TIMESTAMP,
  last_seen       TIMESTAMP,
  utm_source      VARCHAR(100),
  utm_medium      VARCHAR(100),
  utm_campaign    VARCHAR(200),
  referrer        TEXT,
  landing_page    TEXT,
  device_type     VARCHAR(20),   -- mobile/desktop/tablet
  country         VARCHAR(5),
  city            VARCHAR(100),
  pageviews       INT DEFAULT 0,
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Pre-computed funnel snapshots (from PostHog insights API)
CREATE TABLE funnels (
  id              SERIAL PRIMARY KEY,
  date            DATE NOT NULL,
  funnel_name     VARCHAR(50) NOT NULL,  -- e.g. 'visit_to_purchase'
  step            VARCHAR(50) NOT NULL,  -- e.g. 'landing', 'pricing', 'register', 'purchase'
  step_order      SMALLINT NOT NULL,
  count           INT NOT NULL,
  drop_off_pct    DECIMAL(5,2),
  UNIQUE(date, funnel_name, step)
);

-- Bridges PostHog anonymous ID to WP user_id (written on user.registered webhook)
CREATE TABLE conversions (
  id              SERIAL PRIMARY KEY,
  anonymous_id    VARCHAR(100),          -- PostHog ID (nullable: organic direct signup)
  user_id         INT NOT NULL,          -- WP user ID
  registered_at   TIMESTAMP NOT NULL,
  first_purchase_at TIMESTAMP,
  attribution     JSONB,                 -- {utm_source, utm_campaign, referrer, landing_page}
  created_at      TIMESTAMP DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_conversions_user ON conversions(user_id);
CREATE INDEX idx_conversions_anon ON conversions(anonymous_id);
```

### 5.2 Revenue Domain

```sql
-- Attribution source of truth: `conversions` table (Section 5.1).
-- customers.acquisition_source is denormalized FROM conversions at write time.
-- Do not update it independently.
CREATE TABLE customers (
  user_id           INT PRIMARY KEY,
  email             VARCHAR(255),
  display_name      VARCHAR(255),
  first_order_at    TIMESTAMPTZ,
  total_spent       DECIMAL(10,2) DEFAULT 0,
  order_count       INT DEFAULT 0,
  acquisition_source VARCHAR(50),    -- denormalized from conversions.attribution
  first_coupon      VARCHAR(50),
  segment           VARCHAR(20) DEFAULT 'new',  -- new, returning, vip
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
  id                SERIAL PRIMARY KEY,
  wp_order_id       INT UNIQUE NOT NULL,
  user_id           INT NOT NULL,
  amount            DECIMAL(10,2) NOT NULL,
  currency          VARCHAR(3) DEFAULT 'SAR',
  payment_method    VARCHAR(30),     -- mada, visa, mc, apple_pay, tabby
  payment_channel   VARCHAR(30),     -- moyasar, tabby
  status            VARCHAR(20),     -- paid, refunded, failed
  coupon_code       VARCHAR(50),
  discount_pct      DECIMAL(5,2),
  created_at        TIMESTAMP NOT NULL
);

CREATE TABLE order_items (
  id                SERIAL PRIMARY KEY,
  order_id          INT NOT NULL REFERENCES orders(id),
  course_id         INT NOT NULL,
  course_title      VARCHAR(255),
  price             DECIMAL(10,2),       -- final price paid for this item
  discount_amount   DECIMAL(10,2) DEFAULT 0  -- amount discounted from original price (original - price)
);
CREATE INDEX idx_order_items_course ON order_items(course_id);

CREATE TABLE coupons (
  code              VARCHAR(50) PRIMARY KEY,
  type              VARCHAR(20),     -- percentage, fixed
  discount_value    DECIMAL(5,2),
  target_segment    VARCHAR(20),     -- all, new, returning
  total_redemptions INT DEFAULT 0,
  total_revenue     DECIMAL(10,2) DEFAULT 0,
  avg_order_value   DECIMAL(10,2) DEFAULT 0,
  conversion_rate   DECIMAL(5,2),    -- redemptions / views (if tracked)
  first_used        TIMESTAMP,
  last_used         TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE TABLE payment_failures (
  id                SERIAL PRIMARY KEY,
  user_id           INT,
  wp_order_id       INT,
  method            VARCHAR(30),
  channel           VARCHAR(30),
  error_code        VARCHAR(50),
  retried           BOOLEAN DEFAULT false,
  retry_method      VARCHAR(30),
  recovered         BOOLEAN DEFAULT false,
  created_at        TIMESTAMP NOT NULL
);
```

### 5.3 Learning Domain

```sql
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
  UNIQUE(course_id, snapshot_date)
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
  UNIQUE(topic_id, snapshot_date)
);

-- Computed nightly from course_snapshots + feedback + topic_stats.
-- feedback_pct = (up_count / total_count) * 100 from feedback table for that instructor+course.
-- completion_rate = completed_count / enrolled_count from course_snapshots.
-- avg_video_length_sec = AVG(video_duration_sec) from topic_stats for that course.
-- trend_7d = today's feedback_pct - 7 days ago's feedback_pct (NULL if < 7 days of data).
CREATE TABLE instructor_scores (
  id                SERIAL PRIMARY KEY,
  instructor_id     INT NOT NULL,
  instructor_name   VARCHAR(255),
  course_id         INT NOT NULL,
  feedback_pct      DECIMAL(5,2),
  avg_video_length_sec INT,
  completion_rate   DECIMAL(5,2),
  trend_7d          DECIMAL(5,2),   -- change in feedback_pct over 7 days
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
  vote              SMALLINT NOT NULL,  -- 1 = up, -1 = down
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
  created_at        TIMESTAMP NOT NULL
);
CREATE INDEX idx_video_events_topic ON video_events(topic_id, created_at);
```

### 5.4 Retention Domain

```sql
-- Attribution denormalized from conversions table (single source of truth).
CREATE TABLE user_profiles (
  user_id           INT PRIMARY KEY,
  registered_at     TIMESTAMPTZ,
  acquisition_source VARCHAR(50),    -- denormalized from conversions.attribution
  first_course_id   INT,
  total_courses     INT DEFAULT 0,
  current_streak    INT DEFAULT 0,
  longest_streak    INT DEFAULT 0,
  last_active_at    TIMESTAMP,
  status            VARCHAR(20) DEFAULT 'active',  -- active, at_risk, churned
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
  UNIQUE(user_id, date)
);
CREATE INDEX idx_daily_activity_date ON daily_activity(date);

-- Cohort definitions are created manually (admin seeds batch/campaign cohorts).
-- New members are auto-assigned by the User webhook handler:
--   - type='batch': matched via qimah-batch-system enrollment dates
--   - type='campaign': matched via conversions.attribution.utm_campaign
--   - type='organic': users with no campaign attribution
-- Aggregate stats (user_count, avg_retention_30d, churn_rate) recomputed nightly.
CREATE TABLE cohorts (
  id                SERIAL PRIMARY KEY,
  name              VARCHAR(100) NOT NULL,       -- batch_7, jan_campaign, organic_q1
  type              VARCHAR(20) NOT NULL,        -- batch, campaign, organic
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

-- Precomputed nightly from daily_activity: consecutive days with activity
-- working backward from each date. The nightly cron computes this for all active
-- users after daily_activity is populated (3:30 AM). freeze_used comes from
-- gamification webhook data (activity.streak_milestone event payload includes freeze info).
CREATE TABLE streak_history (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  date              DATE NOT NULL,
  streak_count      INT NOT NULL,     -- running count of consecutive study days
  freeze_used       BOOLEAN DEFAULT false,
  UNIQUE(user_id, date)
);
```

### 5.5 Security Domain

```sql
CREATE TABLE sharing_alerts (
  id                SERIAL PRIMARY KEY,
  user_id           INT NOT NULL,
  score             DECIMAL(5,2),
  flags             JSONB,
  status            VARCHAR(20) DEFAULT 'new',  -- new, investigating, cleared, confirmed
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
```

### 5.6 Community Domain

```sql
CREATE TABLE circles (
  id                SERIAL PRIMARY KEY,
  wp_circle_id      INT UNIQUE,
  name              VARCHAR(255),
  member_count      INT DEFAULT 0,
  challenge_type    VARCHAR(30),   -- shared_streak, total_topics, study_days
  completion_rate   DECIMAL(5,2),
  created_at        TIMESTAMP
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
```

### 5.7 Operations Domain

```sql
CREATE TABLE video_pipeline (
  id                SERIAL PRIMARY KEY,
  date              DATE NOT NULL UNIQUE,
  uploads           INT DEFAULT 0,
  processed         INT DEFAULT 0,
  failed            INT DEFAULT 0,
  avg_process_time_sec INT,
  total_storage_gb  DECIMAL(8,2)
);

CREATE TABLE cron_health (
  id                SERIAL PRIMARY KEY,
  job_name          VARCHAR(100) NOT NULL,
  last_run          TIMESTAMP,
  status            VARCHAR(20),     -- ok, warning, failed
  duration_ms       INT,
  error_msg         TEXT,
  updated_at        TIMESTAMP DEFAULT NOW()
);
```

### 5.8 Recruitment Domain

```sql
CREATE TABLE applicants (
  id                SERIAL PRIMARY KEY,
  clickup_id        VARCHAR(50) UNIQUE,
  name              VARCHAR(255),
  email             VARCHAR(255),
  track             VARCHAR(100),     -- discord_mod, community_mgmt, etc.
  status            VARCHAR(50),      -- applied, screening, interview, hired, rejected, ghosted
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
```

### 5.9 Comms Domain

```sql
CREATE TABLE email_campaigns (
  id                SERIAL PRIMARY KEY,
  resend_id         VARCHAR(100) UNIQUE,
  subject           TEXT,
  target_segment    VARCHAR(50),
  sent_count        INT DEFAULT 0,
  open_rate         DECIMAL(5,2),
  click_rate        DECIMAL(5,2),
  bounce_count      INT DEFAULT 0,
  sent_at           TIMESTAMP
);

CREATE TABLE email_events (
  id                SERIAL PRIMARY KEY,
  campaign_id       INT REFERENCES email_campaigns(id),
  user_id           INT,
  email             VARCHAR(255),
  event             VARCHAR(20),     -- delivered, opened, clicked, bounced
  created_at        TIMESTAMP NOT NULL
);
CREATE INDEX idx_email_events_campaign ON email_events(campaign_id);
```

### 5.10 Generic Event Log

```sql
-- Catch-all: every webhook payload lands here (for replay/debugging).
-- Retention: 90 days. Nightly cleanup job deletes rows older than 90 days.
-- Growth estimate: ~5K events/day × 90 days = ~450K rows. At ~1KB avg payload = ~450 MB.
-- If growth exceeds expectations, partition by month:
--   CREATE TABLE events (...) PARTITION BY RANGE (created_at);
--   CREATE TABLE events_2026_03 PARTITION OF events FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
-- For V1, a single table with proper indexing and nightly cleanup is sufficient.
CREATE TABLE events (
  id                SERIAL PRIMARY KEY,
  event_type        VARCHAR(100) NOT NULL,
  user_id           INT,
  payload           JSONB NOT NULL,
  source            VARCHAR(30),     -- wp_webhook, posthog, bunny, resend, clickup
  created_at        TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_user ON events(user_id, created_at);
CREATE INDEX idx_events_created ON events(created_at);  -- for cleanup DELETE WHERE created_at < NOW() - INTERVAL '90 days'
```

### 5.11 Intelligence Layer

```sql
-- Generated by nightly rule engine + weekly LLM
CREATE TABLE insights (
  id                SERIAL PRIMARY KEY,
  section           VARCHAR(20) NOT NULL,   -- revenue, learning, retention, etc.
  severity          VARCHAR(10) NOT NULL,   -- action, watch, win, info
  headline          TEXT NOT NULL,
  body              TEXT NOT NULL,
  evidence          TEXT,                   -- data source attribution
  actions           JSONB,                  -- [{label, url, type: "primary"|"secondary"}]
  rule_id           VARCHAR(50),            -- which rule generated this (null for AI)
  is_ai             BOOLEAN DEFAULT false,
  is_pinned         BOOLEAN DEFAULT false,
  is_dismissed      BOOLEAN DEFAULT false,
  expires_at        TIMESTAMP,
  created_at        TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_insights_section ON insights(section, severity, created_at);
CREATE INDEX idx_insights_active ON insights(is_dismissed, expires_at);

-- Near-real-time key metrics (updated by n8n on each relevant webhook)
CREATE TABLE pulse (
  section           VARCHAR(20) NOT NULL,
  key               VARCHAR(50) NOT NULL,   -- 'online_now', 'revenue_today', etc.
  value             NUMERIC,                -- raw number (e.g., 12500, 84.5)
  label             VARCHAR(100),
  delta             VARCHAR(50),            -- display-ready string: '+8%', '-31% vs last week'. VARCHAR because deltas mix formats (%, absolute, text qualifiers). n8n computes and formats on write.
  sort_order        SMALLINT DEFAULT 0,
  updated_at        TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY(section, key)
);

-- Cross-section summary (updated by nightly job)
CREATE TABLE summary (
  id                SERIAL PRIMARY KEY,
  date              DATE NOT NULL UNIQUE,
  action_count      INT DEFAULT 0,
  watch_count       INT DEFAULT 0,
  win_count         INT DEFAULT 0,
  computed_at       TIMESTAMP DEFAULT NOW()
);
```

**Total: 33 tables across 11 groups.**

---

## 6. Insight Engine (Rule-Based)

### Architecture

A single script (Node.js or Python) runs via cron at **3:00 AM AST daily**. It:

1. Reads from domain tables (yesterday's snapshots, last 7 days of events)
2. Evaluates rules
3. Writes insight rows to the `insights` table
4. Updates the `summary` table
5. Archives expired insights (soft-delete via `is_dismissed`)

### Rule Format

Each rule is a function that queries Postgres and returns zero or more insight objects:

```javascript
// Example rule: instructor feedback drop
{
  id: 'learning.instructor_feedback_drop',
  section: 'learning',
  schedule: 'daily',
  async evaluate(db) {
    const drops = await db.query(`
      SELECT instructor_id, instructor_name, course_id,
             feedback_pct, trend_7d
      FROM instructor_scores
      WHERE snapshot_date = CURRENT_DATE - 1
        AND trend_7d < -10
    `);

    return drops.map(d => ({
      severity: 'action',
      headline: `${d.instructor_name}: feedback dropped ${Math.abs(d.trend_7d)} points this week`,
      body: `...`, // enriched with comment analysis
      evidence: 'Source: CEP feedback + instructor_scores',
      actions: [
        { label: 'View feedback', url: `/learning/instructor/${d.instructor_id}`, type: 'primary' },
        { label: 'Message instructor', url: `#message:${d.instructor_id}`, type: 'secondary' }
      ]
    }));
  }
}
```

### Rule Inventory (V1)

| Rule ID | Section | Severity | Trigger |
|---------|---------|----------|---------|
| `revenue.payment_failures_spike` | Revenue | watch | Payment failures > 1.5x 4-week avg |
| `revenue.coupon_segment_insight` | Revenue | info | Enough data (50+ redemptions) to compare segments |
| `revenue.bundle_opportunity` | Revenue | watch | Course co-purchase overlap > 60% |
| `revenue.revenue_anomaly` | Revenue | watch | Week-over-week revenue change > 25% |
| `revenue.roas` | Revenue | info | Campaign with enough conversions to compute ROAS |
| `learning.instructor_feedback_drop` | Learning | action | Instructor feedback drops > 10 points in 7 days |
| `learning.course_completion_change` | Learning | win/watch | Completion rate changes > 10% after content update |
| `learning.video_length_analysis` | Learning | info | Enough data to compute optimal video length |
| `learning.negative_feedback_cluster` | Learning | action | 3+ negative feedback entries on same topic in 7 days |
| `retention.inactive_students` | Retention | action | Previously active students with 0 activity for N days |
| `retention.cohort_churn_comparison` | Retention | watch | Campaign cohort churn > 2x organic cohort |
| `retention.at_risk_detection` | Retention | watch | Activity declining pattern (sessions dropping 3 consecutive days) |
| `security.sharing_alert` | Security | action | New sharing detection flag |
| `security.device_anomaly` | Security | watch | Device switches > 3x user average |
| `community.challenge_effectiveness` | Community | info | Compare completion rates across challenge types |
| `ops.pipeline_failure` | Operations | action | Video processing failures > 0 |
| `ops.cron_failure` | Operations | action | Any cron job in failed state |
| `recruitment.sla_breach` | Recruitment | action | Applicants pending > SLA threshold (3 days) |
| `recruitment.ghost_pattern` | Recruitment | watch | Track-level ghost rate > 20% |
| `comms.open_rate_anomaly` | Comms | watch | Open rate deviates > 15% from baseline |

### Deduplication & Severity Escalation

Rules check if an active (not dismissed, not expired) insight with the same `rule_id` and matching key data already exists before inserting. Prevents "12 students inactive" from appearing every day until resolved.

Update strategy: if existing insight found:
- Update body/evidence with fresh data
- Keep original `created_at` (shows how long the issue has persisted)
- **Severity can escalate** (watch -> action) but **never demote** (action -> watch). If the underlying data worsens, the insight upgrades. The original concern remains valid even if the number improves slightly.

### Engine Self-Monitoring

The insight engine writes a `pulse` row on successful completion:
```sql
INSERT INTO pulse (section, key, value, label, updated_at)
VALUES ('system', 'engine_last_run', 1, 'Insight Engine', NOW())
ON CONFLICT (section, key) DO UPDATE SET updated_at = NOW();
```

The dashboard shows a warning banner if `engine_last_run` is more than 26 hours old ("Insight engine has not run since [time] - data may be stale"). On engine failure, a notification email is sent via Resend.

---

## 7. Weekly AI Analysis

### Schedule

Every **Sunday at 4:00 AM AST** (before the weekly meeting).

### Process

1. Aggregate the week's data into a structured context document:
   - Revenue: total, trend, top coupons, bundle overlaps
   - Learning: per-instructor feedback scores, completion rates, video stats
   - Retention: DAU/WAU/MAU, cohort churn rates, inactive clusters
   - Active insights from the week (what the rule engine flagged)
2. Send to Claude API (claude-sonnet-4-6 for cost efficiency) with a system prompt:
   - "You are an analytics advisor for Qimah, an Arabic educational platform..."
   - "Produce 2-3 insights per section that go beyond what rule-based alerts would catch"
   - "Focus on cross-domain patterns, root cause analysis, and strategic recommendations"
   - "Output structured JSON with section, headline, body fields"
3. Parse response, write to `insights` table with `is_ai = true`
4. One AI block per section, plus a "Cross-Section" summary

### Cost Estimate

~10K input tokens (aggregated data) + ~2K output tokens per run.
At Sonnet pricing: ~$0.04 per weekly run. Negligible.

### Prompt Structure

```
You are an analytics advisor for Qimah, an online education platform in Saudi Arabia.

Here is this week's data summary:
{structured data}

Here are the rule-based alerts that fired this week:
{list of active insights}

For each of the 9 sections (Revenue, Learning, Retention, Pulse, Security,
Community, Operations, Recruitment, Comms), produce 0-2 insights that:
1. Identify patterns the rules missed (cross-domain correlations, root causes)
2. Suggest specific actions with reasoning
3. Are grounded in the data provided - never fabricate numbers

Also produce 1-2 cross-section insights that connect patterns across domains.

Output as JSON array of {section, headline, body} objects.
```

---

## 8. n8n Workflows

### 8.1 Webhook Receiver (real-time)

One n8n workflow per webhook category. Each:
1. Receives POST from WP webhook
2. Validates HMAC signature (`X-Qimah-Signature`)
3. Writes raw event to `events` table
4. Writes to domain-specific table (e.g., `orders`, `feedback`)
5. Updates relevant `pulse` row

| Workflow | Trigger | Writes to |
|----------|---------|-----------|
| Revenue webhook | commerce.order_paid, commerce.order_refunded | orders, order_items, customers, coupons (upsert aggregate stats), pulse |
| Activity webhook | activity.logged, topic.completed | daily_activity, pulse |
| Session webhook | session.started | daily_activity, pulse |
| User webhook | user.registered | user_profiles, conversions, pulse |
| Points webhook | points.awarded | daily_activity |
| Streak webhook | activity.streak_milestone | streak_history |
| Security webhook | Various sec events | sharing_alerts, pulse |
| Community webhook | circle.* | circles, circle_activity (discord_activity deferred to V2 - requires bot) |
| Feedback webhook | feedback.submitted | feedback (n8n transforms: rating string→vote int, derives instructor_id from course author, fetches comment text from comment_id) |
| Booking webhook | booking.* | events (generic for V1) |

### 8.2 API Pullers (nightly via cron)

| Workflow | Source | Schedule | Writes to |
|----------|--------|----------|-----------|
| WP Enrollments | GET /analytics/enrollments + GET /wp/v2/sfwd-courses?_fields=id,author | 3:00 AM | course_snapshots (incl. instructor_id from course author), topic_stats |
| WP Feedback | GET /analytics/feedback | 3:00 AM | course_snapshots (feedback_pct) |
| WP Leaderboard | GET /leaderboard | 3:00 AM | user_profiles (streak data) |
| Moyasar Failures | Moyasar API (payments?status=failed) | 3:00 AM | payment_failures |
| Bunny Stats | Bunny API + WP video-manager REST if needed | 3:00 AM | video_pipeline, topic_stats (duration) |
| Resend Stats | Resend API | 3:00 AM | email_campaigns, email_events |
| ClickUp Pipeline | ClickUp API | 3:00 AM | applicants, pipeline_snapshots |
| PostHog Sync | PostHog API | 3:30 AM | visitors, funnels, conversions, video_events (Bunny iframe engagement) |
| Instructor Scores | Computed from snapshots | 3:30 AM | instructor_scores |
| Streak Computation | Computed from daily_activity | 3:30 AM | streak_history, user_profiles (current_streak, longest_streak) |
| Cohort Stats | Computed from cohort_members + daily_activity | 3:30 AM | cohorts (user_count, avg_retention_30d, churn_rate) |
| Security Patterns | Computed from events (security webhooks, 7d window) | 3:30 AM | session_patterns |
| Cron Health | n8n self-reports after each workflow | continuous | cron_health (each workflow upserts its own row on completion/failure) |

### 8.3 Insight Engine (nightly)

| Workflow | Schedule | Does |
|----------|----------|------|
| Rule Evaluator | 4:00 AM | Runs all rules, writes insights |
| AI Weekly | Sunday 4:00 AM | Sends data to Claude, writes AI insights |
| Cleanup | 5:00 AM | Archives expired insights, deletes events older than 90 days (`DELETE FROM events WHERE created_at < NOW() - INTERVAL '90 days'`), runs `VACUUM ANALYZE events` to reclaim space |

---

## 9. Next.js Dashboard App

### Tech Stack

- **Next.js 15** (App Router)
- **React 19**
- **Tailwind CSS** (RTL via `dir="rtl"`)
- **Server Components** for data fetching (no client-side Postgres queries)
- **Server Actions** for dismiss/pin operations

### Route Structure

```
/                       → Redirect to /revenue (or last visited section)
/[section]              → Section page (revenue, learning, retention, etc.)
/api/pulse              → Live pulse data (polled every 30s by client)
/api/insights/[section] → Insights for a section
/api/summary            → Cross-section summary counts
```

### Data Fetching

- **Pulse strip:** Client-side polling every 30 seconds via `/api/pulse?section=revenue`
- **Insight cards:** Server-rendered on page load, no client polling (insights change daily, not in real-time)
- **Summary bar:** Server-rendered, revalidated every 5 minutes via ISR

### Auth

Simple: HTTP Basic Auth or a shared token. Admin + team only, no public access. Can add proper auth (NextAuth) in V2 if needed. Dashboard is on a private VPS, accessible via Tailscale or VPN.

### Dismissing / Pinning Insights

- Dismiss: sets `is_dismissed = true`, removes from view
- Pin: sets `is_pinned = true`, prevents auto-expiry
- Both via Server Actions (POST to Postgres, revalidate page)

---

## 10. PostHog Configuration

### What to Track (JS snippet on qimah.net)

PostHog's autocapture handles most events. Explicitly define:

| Event | Where | Why |
|-------|-------|-----|
| `$pageview` | All pages | Autocapture (default) |
| `pricing_viewed` | /pricing page | Funnel step |
| `signup_started` | Registration form focus | Funnel step |
| `signup_completed` | user.registered (server-side) | Funnel completion + user identification |
| `first_purchase` | commerce.order_paid (server-side) | Revenue attribution |

### Funnels to Define

1. **Visit → Purchase:** Landing → Pricing → Register → First Purchase
2. **Register → Active:** Register → First Login → First Topic → Day 7 Active

### Session Replay

Enable for anonymous visitors only (pre-login). Disable for logged-in students (privacy + storage). Configure: 1% sample rate (sufficient for funnel debugging, saves storage).

### User Identification

On `user.registered` webhook, n8n calls PostHog `identify` API to merge `anonymous_id` with `user_id`. This enables cross-device attribution and powers the `conversions` bridge table.

---

## 11. Phased Rollout

### Phase 1: Foundation (Week 1-2)
- Postgres schema deployed
- n8n webhook receivers for: commerce, activity, session, user, feedback
- PostHog installed + JS snippet on qimah.net
- WP: implement analytics API spec (2 endpoints + read:analytics scope + user.registered webhook)
- Pulse table populated in real-time
- Dashboard skeleton: 9 tabs, pulse strips, empty insight sections

### Phase 2: First Insights (Week 3-4)
- n8n nightly pullers for: WP enrollments/feedback, Bunny, Resend, ClickUp
- Snapshot tables populating daily
- Rule engine V1: 10 highest-value rules (inactive students, instructor feedback, payment failures, SLA breach, revenue anomaly)
- Dashboard shows real insight cards

### Phase 3: Intelligence (Week 5-6)
- PostHog sync workflow (visitors, funnels, conversions)
- Remaining rules implemented (20 total)
- Weekly AI analysis pipeline
- Full dashboard operational

### Phase 4: Polish (Week 7-8)
- Wire remaining dead WP webhooks (~15 events, see analytics API spec)
- Cohort management UI
- Insight dismiss/pin workflow
- Dashboard performance optimization
- Team onboarding

---

## 12. Out of Scope (V2)

- Instructor-scoped views (data model supports it, UI doesn't)
- Custom rule builder (admin creates rules via UI)
- Alerting (Slack/email notifications when action-severity insight fires)
- Mobile-optimized dashboard
- Historical trend charts (V1 is insight-first, charts are V2)
- Student-facing analytics
- A/B test integration with PostHog feature flags
- Automated actions (auto-send re-engagement email when rule fires)

---

## 13. Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Dashboard framework | Next.js | Already used for FMS, team knows it, SSR for fast loads |
| Insight storage | Postgres table, not in-memory | Persist across restarts, queryable, auditable |
| Rule engine | Simple JS functions, not a rules DSL | 20 rules don't justify a framework. Functions are debuggable. |
| AI model | Sonnet (not Opus) for weekly | Cost: $0.04/week. Sonnet handles structured analysis well. Upgrade if quality insufficient. |
| PostHog sync | Nightly via API, not real-time | Pre-login data doesn't need real-time. Funnels are daily metrics. |
| Auth | Basic/token, not NextAuth | 3-5 users on a VPN. Full auth is overhead for V1. |
| Schema | Domain tables, not generic events | Rules need fast, indexed queries. Generic JSONB queries are slow for analytics. |
| Pulse updates | n8n writes on each webhook | Sub-minute freshness without a separate streaming system. |
| Insight expiry | 7-14 days with pin option | Prevents stale insights cluttering the view. Pinning preserves important ones. |
| Summary bar | Cross-section counts | One glance tells you if today is a 0-action day or a 5-action day. |
