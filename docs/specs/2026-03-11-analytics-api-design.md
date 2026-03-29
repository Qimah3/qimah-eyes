# Analytics API Layer — Design Spec

**Date:** 2026-03-11
**Plugin:** qimah-profile
**Branch:** `feature/analytics-api`
**Status:** Draft (rev 2 — post spec review)

## Summary

Add a thin read-only REST API layer to qimah-profile that exposes snapshot data the VPS analytics server needs but can't get from live webhook events. Two endpoints under `/qimah/v1/analytics/` (enrollments + feedback), authenticated via existing API key system with a new `read:analytics` scope. Leaderboard data uses the existing `/leaderboard` endpoint. Also adds and wires the missing `user.registered` webhook event.

## Context & Architecture

Qimah is building a centralized analytics platform on a dedicated VPS (Hetzner CPX41, 8 vCPU / 16 GB). The VPS runs:

- **PostHog** (self-hosted) — pre-login analytics, funnels, session replays, autocapture
- **n8n** — ingestion layer, pulls WP API + receives webhooks, writes to Postgres
- **Postgres** — analytics DB for all post-login data
- **Custom Next.js dashboard** — GOD dashboard with 9 sections, instructor scoping, RTL Arabic
- **FMS** — Find My Schedule app (already exists)

**Data flow:**

```
WP webhooks (live events) ──→ n8n ──→ Postgres ──→ Dashboard
WP REST API (snapshots)   ──→ n8n ──→ Postgres ──↗
Bunny API ─────────────────→ n8n ──→ Postgres ──↗
Resend API ────────────────→ n8n ──→ Postgres ──↗
ClickUp API ───────────────→ n8n ──→ Postgres ──↗
```

WordPress is one data source among many. It serves raw data only — all aggregation happens on the VPS. The VPS pulls external APIs (Bunny, Resend, ClickUp) directly without routing through WP.

### Data Sources (22 total, 9 dashboard sections)

| Section | Sources | Delivery |
|---------|---------|----------|
| **Pulse** (real-time) | Sessions, Peak Lights | Webhook push |
| **Revenue** | WooCommerce, Moyasar, QCX, invoices | Webhook push |
| **Learning** | LearnDash, Bunny, feedback, QGami | Webhook push + API snapshot |
| **Retention** | Sessions (DAU/WAU/MAU), streaks | Webhook push + API snapshot |
| **Security** | qimah-sec, session logs | Webhook push |
| **Community** | Circles, Discord, calendar | Webhook push |
| **Operations** | Video pipeline, cron health | Webhook push |
| **Recruitment** | Tally, ClickUp | VPS pulls directly |
| **Comms** | Resend | VPS pulls directly |

### Why These Three Endpoints

Most data reaches the VPS via live webhook events (50+ event types already defined). These endpoints exist for data that has **no natural event moment** — point-in-time snapshots:

| Endpoint | Why webhooks can't cover it |
|----------|---------------------------|
| Enrollments | Progress % changes gradually, no discrete event fires |
| Feedback | `feedback.submitted` exists for live, but aggregate stats need a snapshot |

**Note:** Leaderboard was considered but dropped — the existing `GET /leaderboard` and `GET /leaderboard/course/{id}` endpoints (scope `read:leaderboard`) already serve this data. The VPS can use those directly with a key that has `read:leaderboard` scope. Adding `streak_days` to the existing endpoint (if needed) is a smaller change than a duplicate endpoint.

Backfill of historical data is handled via direct DB queries (SSH / Postgres MCP), not through these endpoints.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Endpoint count | 2 (not 6+) | Webhooks handle live events; only snapshot data needs pull endpoints. Leaderboard dropped — existing `/leaderboard` endpoint covers it |
| Location | New file in profile's `/api/` dir | Keeps analytics separate from core REST API class |
| Auth | Existing API key + new `read:analytics` scope | No new auth system; scope prevents existing keys from accessing analytics |
| Aggregation | None — serve raw data | VPS does all computation; WP stays dumb and fast |
| Pagination | Per-user within course (enrollments) | LearnDash progress queries are expensive (serialized user meta) |
| Backfill | Direct DB query, not API | One-time operation, no need to build endpoint infrastructure |
| Instructor scoping | Not in this spec | Requires auth design work; admin-only for now |
| Dead webhook wiring | Separate PR | ~20 unwired events exist; different scope, different risk |

## Endpoints

### Auth

All endpoints require `X-Qimah-API-Key` header with `read:analytics` scope. Returns `403` if scope missing.

Rate limit: 30 requests/minute (analytics queries can be heavy). Must add `'read:analytics' => 30` to `RATE_LIMITS` constant in `class-qimah-api-keys.php`.

### Error Responses

All endpoints return standard WP REST error format:

```json
{"code": "error_code", "message": "Human-readable message", "data": {"status": 4xx}}
```

| Code | Status | When |
|------|--------|------|
| `rest_forbidden` | 401 | Missing or invalid API key |
| `insufficient_scope` | 403 | Key lacks `read:analytics` scope |
| `rate_limit_exceeded` | 429 | Over 30 requests/minute |
| `course_not_found` | 404 | `course_id` is not a published LearnDash course |
| `learndash_not_active` | 200 | LearnDash deactivated — returns `{"courses": [], "error": "learndash_not_active"}` |
| `cep_not_active` | 200 | CEP deactivated (feedback endpoint) — returns `{"courses": []}` with empty stats |

### `GET /qimah/v1/analytics/enrollments`

Returns enrolled users with progress percentage per course.

**Parameters:**

| Param | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `course_id` | int | No | all courses | Filter to single course |
| `page` | int | No | 1 | Page number |
| `per_page` | int | No | 50 | Max 250 (high ceiling for automated n8n sync) |

**Response (single course):**

```json
{
  "course_id": 123,
  "title": "Course Name",
  "enrolled_count": 84,
  "completed_count": 12,
  "users": [
    {
      "user_id": 456,
      "progress": 73,
      "completed": false,
      "enrolled_at": "2025-09-01T00:00:00+03:00",
      "last_active": "2026-03-10T14:30:00+03:00"
    }
  ],
  "page": 1,
  "total_pages": 2,
  "total_users": 84
}
```

**Response (all courses, no `course_id`):**

```json
{
  "courses": [
    {
      "course_id": 123,
      "title": "Course Name",
      "enrolled_count": 84,
      "completed_count": 12
    }
  ],
  "total_courses": 8
}
```

When listing all courses, user-level detail is omitted (too heavy). Use `course_id` param to drill down.

**Field notes:**
- `progress`: integer 0–100 (rounded from LearnDash float via `round()`)
- `enrolled_at`: sourced from `get_user_meta($uid, 'course_{cid}_access_from', true)` — LearnDash stores Unix timestamp when access is granted. Returns `null` if missing (legacy enrollments before LearnDash tracked this). **Not** the batch system date.
- `last_active`: sourced from `Qimah_Step_Visits::get_course_last_active($uid, $cid)` — last topic/lesson view timestamp. Returns `null` if no activity recorded.

**Implementation:**
- Guard: `function_exists('learndash_get_users_for_course')` — return `learndash_not_active` error if missing
- `learndash_get_users_for_course($course_id)` for enrolled users
- `learndash_course_progress(['user_id' => $uid, 'course_id' => $cid, 'array' => true])` for progress %
- `update_postmeta_cache()` / `update_meta_cache('user', $ids)` to avoid N+1
- Transient cache: `qimah_analytics_enrollment_{course_id}_p{page}` with 5-minute TTL (per-page cache keys)

### `GET /qimah/v1/analytics/feedback`

Returns aggregated feedback stats (thumbs up/down) per course.

**Parameters:**

| Param | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `course_id` | int | No | all courses | Filter to single course |

**Response:**

```json
{
  "courses": [
    {
      "course_id": 123,
      "title": "Course Name",
      "total": 156,
      "up": 132,
      "down": 24,
      "percentage": 85,
      "recent_issues": [
        {
          "topic_id": 789,
          "topic_title": "Topic Name",
          "comment": "الصوت مو واضح",
          "date": "2026-03-10T14:30:00+03:00"
        }
      ]
    }
  ]
}
```

**Field notes:**
- `recent_issues`: last 3 negative feedback entries (thumbs down) with non-empty comments, no time window — matches existing `get_course_feedback_stats()` behavior
- Feedback and all-courses listing do not paginate because course count (~8) is bounded

**Implementation:**
- Reuses `Qimah_CEP_Feedback::get_course_feedback_stats($course_id)` (static, public)
- Guarded with `class_exists('Qimah_CEP_Feedback')` — returns empty array if CEP not active
- All courses: `get_posts(['post_type' => 'sfwd-courses', 'post_status' => 'publish', 'numberposts' => -1, 'fields' => 'ids'])`
- Transient cache: `qimah_analytics_feedback_{course_id}` with 10-minute TTL

### Leaderboard — Uses Existing Endpoint

**Not a new endpoint.** The existing `GET /qimah/v1/leaderboard` and `GET /qimah/v1/leaderboard/course/{id}` endpoints already serve points/ranking data via `QGami_Points_Mirror`. The VPS uses these with a key that has `read:leaderboard` scope.

If streak data is needed in the leaderboard response, that's a minor enhancement to the existing endpoint (add `?include=streaks` param) — not a new analytics endpoint.

## New Webhook Event: `user.registered`

**Event key:** `user.registered`
**Description:** Fired when a new user account is created.
**Hook:** `user_register` (WordPress core)

**Payload:**
```json
{
  "user_id": 456,
  "email": "student@example.com",
  "display_name": "Student Name",
  "registered_at": "2026-03-11T10:00:00+03:00",
  "roles": ["subscriber"]
}
```

**Handler:** `on_user_registered($user_id)` — reads user data from `get_userdata()`, dispatches event.

**Placement:** Add to the Admin events group in `EVENTS` constant, before existing `user.banned`/`user.unbanned` entries. Add `add_action('user_register', [$this, 'on_user_registered'], 10, 1)` in the constructor alongside other hook listeners.

## New API Key Scope: `read:analytics`

Added to the valid scopes list in `class-qimah-api-keys.php`. Existing keys without this scope cannot access `/analytics/*` endpoints.

**Scope definition:**
```php
'read:analytics' => 'Read-only access to analytics snapshot endpoints'
```

## Dead Webhook Events (Separate Task)

The following events are defined in `Qimah_Webhooks::EVENTS` but have **no handler method or `add_action()` listener wired** — they appear in the Admin Hub event list but never actually fire. They should be wired in a separate PR to enable full live data flow to the VPS:

**Priority (needed for dashboard):**
- `session.started` — DAU tracking
- `commerce.order_paid` / `commerce.order_refunded` — revenue analytics
- `booking.created` / `booking.confirmed` / `booking.completed` / `booking.no_show` — instructor utilization
- `circle.created` / `circle.member_joined` / `circle.member_left` / `circle.challenge_completed` — community

**Lower priority:**
- `course.started` — could derive from first `topic.completed`
- `user.banned` / `user.unbanned` — admin events, low volume
- `commerce.subscription_*` — no subscriptions currently
- `commerce.coupon_applied` / `cart_abandoned` — QCX integration
- `commerce.course_access_granted` / `commerce.course_access_revoked` — derivable from order events
- `discord.roles_synced` — low value for analytics
- `points.milestone` — derivable from points.awarded

## VPS Infrastructure Reference

Documented here for context; implementation is out of scope for this spec.

**Server:** Hetzner CPX41 (8 vCPU, 16 GB RAM, 240 GB NVMe, ~€28/mo)

| Service | Role | Estimated RAM |
|---------|------|---------------|
| PostHog (self-hosted) | Pre-login analytics, funnels, session replays | 5-6 GB |
| n8n | Ingestion — pulls WP/Bunny/Resend APIs, receives webhooks | 512 MB |
| Postgres | Analytics DB | 1 GB |
| Dashboard (Next.js) | Custom GOD dashboard, 9 sections, instructor scoping | 256 MB |
| FMS (Next.js) | Find My Schedule | 256 MB |
| OS + headroom | | 2 GB |

**Dashboard sections:** Pulse, Revenue, Learning, Retention, Security, Community, Operations, Recruitment, Comms

## File Changes

| File | Change |
|------|--------|
| `qimah-profile/includes/api/class-qimah-rest-analytics.php` | **New file** — 2 GET endpoints, scope check, caching |
| `qimah-profile/includes/api/class-qimah-rest-api.php` | Require + instantiate analytics class |
| `qimah-profile/includes/api/class-qimah-api-keys.php` | Add `read:analytics` to valid scopes + `RATE_LIMITS` entry (30/min) |
| `qimah-profile/includes/api/class-qimah-webhooks.php` | Add `user.registered` to EVENTS + handler + hook listener |
| `qimah-profile/qimah-profile.php` | Version bump (minor — new feature) |

## Out of Scope

- VPS setup, Docker, n8n workflows, Postgres schema
- Wiring dead webhook events (separate PR, listed above)
- Instructor-scoped API access (needs auth design — later)
- Bunny/Resend/ClickUp ingestion (VPS pulls directly)
- PostHog configuration
- Next.js dashboard app
- Historical data backfill (direct DB queries)
- Write endpoints (analytics is read-only)
