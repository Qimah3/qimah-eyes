# Analytics API Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 2 read-only REST endpoints (`/analytics/enrollments`, `/analytics/feedback`), a new `read:analytics` API key scope, and the `user.registered` webhook event.

**Architecture:** New `class-qimah-rest-analytics.php` in profile's API dir, following the same singleton + `rest_api_init` pattern as `class-qimah-rest-calendar.php`. Scope/rate-limit added to `class-qimah-api-keys.php`. Webhook event added to `class-qimah-webhooks.php`. All changes in qimah-profile only.

**Tech Stack:** PHP, WordPress REST API, LearnDash API, WooCommerce

**Spec:** `docs/superpowers/specs/2026-03-11-analytics-api-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|---------------|
| `qimah-profile/includes/api/class-qimah-rest-analytics.php` | **Create** | 2 GET endpoints with scope check, pagination, transient caching |
| `qimah-profile/includes/api/class-qimah-api-keys.php` | Modify (lines 49, 79) | Add `read:analytics` scope + 30/min rate limit |
| `qimah-profile/includes/api/class-qimah-webhooks.php` | Modify (EVENTS const + constructor + handlers) | Add `user.registered` event definition + hook + handler |
| `qimah-profile/qimah-profile.php` | Modify (lines 185-188, 6, 68, 205) | `require_once` for new file + version bump (minor: 6.23.0) |

---

## Chunk 1: Scope, Rate Limit, and Webhook Event

### Task 1: Add `read:analytics` scope and rate limit

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-api-keys.php:49` (SCOPES) and `:79` (RATE_LIMITS)

- [ ] **Step 1: Add scope to SCOPES constant**

In `class-qimah-api-keys.php`, insert after `'read:calendar'` (line 49):

```php
        'read:analytics'       => 'Read-only access to analytics snapshot endpoints',
```

- [ ] **Step 2: Add rate limit to RATE_LIMITS constant**

In `class-qimah-api-keys.php`, insert after `'read:calendar' => 60,` (line 79):

```php
        'read:analytics'       => 30,
```

- [ ] **Step 3: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-api-keys.php
git commit -m "feat(api): add read:analytics scope with 30/min rate limit"
```

---

### Task 2: Add `user.registered` webhook event + handler

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php`

- [ ] **Step 1: Add event to EVENTS constant**

Insert before the `// Admin events` comment and `'user.banned'` entry (line ~70):

```php
        // User lifecycle events
        'user.registered'           => 'Fired when a new user account is created',

```

- [ ] **Step 2: Add hook registration in constructor**

Insert after the Qimah Eyes pipeline block (after `woocommerce_applied_coupon` line) and before `// CEP feedback events`:

```php
        // User lifecycle
        add_action( 'user_register', [ $this, 'on_user_registered' ], 10, 1 );
```

- [ ] **Step 3: Add handler method**

Insert after the `on_coupon_applied` method (before `format_wc_order`):

```php
    /**
     * Handle user registered event
     *
     * @param int $user_id New user ID
     */
    public function on_user_registered( $user_id ) {
        $user = get_userdata( $user_id );
        if ( ! $user ) {
            return;
        }

        $this->dispatch( 'user.registered', [
            'user_id'       => $user_id,
            'email'         => $user->user_email,
            'display_name'  => $user->display_name,
            'registered_at' => mysql2date( 'c', $user->user_registered ),
            'roles'         => $user->roles,
        ] );
    }
```

- [ ] **Step 4: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): add user.registered event

Hooks user_register, dispatches with email, display_name, roles."
```

---

## Chunk 2: Analytics REST Controller

### Task 3: Create `class-qimah-rest-analytics.php`

**Files:**
- Create: `qimah-profile/includes/api/class-qimah-rest-analytics.php`

This is the main task. The class follows `class-qimah-rest-calendar.php`'s pattern: singleton, `rest_api_init`, own `check_api_key_scope` method.

- [ ] **Step 1: Create the file with class skeleton + permission check + route registration**

Create `qimah-profile/includes/api/class-qimah-rest-analytics.php`:

```php
<?php
/**
 * Qimah REST Analytics API
 *
 * Read-only snapshot endpoints for the Qimah Eyes analytics pipeline.
 * Data that has no natural webhook event moment (enrollment progress, feedback aggregates).
 *
 * Endpoints:
 * - GET /analytics/enrollments  - Enrolled users with progress per course
 * - GET /analytics/feedback     - Aggregated feedback stats per course
 *
 * @package    The_Qimah
 * @subpackage API
 * @since      6.23.0
 */

if ( ! defined( 'ABSPATH' ) ) exit;

class Qimah_REST_Analytics {

    /** @var string */
    const NAMESPACE = 'qimah/v1';

    /** @var Qimah_REST_Analytics */
    private static $instance = null;

    /**
     * Get singleton instance
     */
    public static function get_instance() {
        if ( null === self::$instance ) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    /**
     * Constructor
     */
    private function __construct() {
        add_action( 'rest_api_init', [ $this, 'register_routes' ] );
    }

    /**
     * Register analytics routes
     */
    public function register_routes() {
        // GET /analytics/enrollments
        register_rest_route( self::NAMESPACE, '/analytics/enrollments', [
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => [ $this, 'get_enrollments' ],
            'permission_callback' => [ $this, 'check_permission' ],
            'args'                => [
                'course_id' => [
                    'type'              => 'integer',
                    'sanitize_callback' => 'absint',
                ],
                'page' => [
                    'type'              => 'integer',
                    'default'           => 1,
                    'sanitize_callback' => 'absint',
                ],
                'per_page' => [
                    'type'              => 'integer',
                    'default'           => 50,
                    'sanitize_callback' => 'absint',
                ],
            ],
        ] );

        // GET /analytics/feedback
        register_rest_route( self::NAMESPACE, '/analytics/feedback', [
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => [ $this, 'get_feedback' ],
            'permission_callback' => [ $this, 'check_permission' ],
            'args'                => [
                'course_id' => [
                    'type'              => 'integer',
                    'sanitize_callback' => 'absint',
                ],
            ],
        ] );
    }

    // =========================================================================
    // PERMISSION
    // =========================================================================

    /**
     * Check read:analytics permission
     *
     * API key with read:analytics scope required. Admins pass through.
     *
     * @param WP_REST_Request $request
     * @return bool|WP_Error
     */
    public function check_permission( $request ) {
        if ( current_user_can( 'manage_options' ) ) {
            return true;
        }

        $api_key = $request->get_header( 'X-Qimah-API-Key' );
        if ( ! $api_key ) {
            return new WP_Error(
                'rest_forbidden',
                'API key required.',
                [ 'status' => 401 ]
            );
        }

        if ( ! class_exists( 'Qimah_API_Keys' ) ) {
            return new WP_Error(
                'rest_forbidden',
                'API key system not available.',
                [ 'status' => 500 ]
            );
        }

        $result = Qimah_API_Keys::authenticate_request( $request, 'read:analytics' );
        if ( is_wp_error( $result ) ) {
            return $result;
        }

        return true;
    }

    // =========================================================================
    // ENROLLMENTS
    // =========================================================================

    /**
     * GET /analytics/enrollments
     *
     * Without course_id: returns course summary list (no user detail).
     * With course_id: returns paginated user-level enrollment data.
     *
     * @param WP_REST_Request $request
     * @return WP_REST_Response
     */
    public function get_enrollments( $request ) {
        if ( ! function_exists( 'learndash_get_users_for_course' ) ) {
            return new WP_REST_Response( [
                'courses' => [],
                'error'   => 'learndash_not_active',
            ], 200 );
        }

        $course_id = $request->get_param( 'course_id' );

        if ( $course_id ) {
            return $this->get_course_enrollments( $course_id, $request );
        }

        return $this->get_all_courses_summary();
    }

    /**
     * Get summary of all published courses with enrollment/completion counts
     *
     * @return WP_REST_Response
     */
    private function get_all_courses_summary() {
        $cache_key = 'qimah_analytics_enrollment_summary';
        $cached    = get_transient( $cache_key );
        if ( false !== $cached ) {
            return new WP_REST_Response( $cached, 200 );
        }

        $courses = get_posts( [
            'post_type'   => 'sfwd-courses',
            'post_status' => 'publish',
            'numberposts' => -1,
            'fields'      => 'ids',
        ] );

        $data = [];
        foreach ( $courses as $course_id ) {
            $users = learndash_get_users_for_course( $course_id, [], false );
            $user_ids = $users instanceof WP_User_Query ? $users->get_results() : [];
            $enrolled_count = count( $user_ids );

            $completed_count = 0;
            foreach ( $user_ids as $uid ) {
                $progress = learndash_course_progress( [
                    'user_id'   => $uid,
                    'course_id' => $course_id,
                    'array'     => true,
                ] );
                if ( ! empty( $progress['percentage'] ) && (int) $progress['percentage'] === 100 ) {
                    $completed_count++;
                }
            }

            $data[] = [
                'course_id'       => $course_id,
                'title'           => get_the_title( $course_id ),
                'enrolled_count'  => $enrolled_count,
                'completed_count' => $completed_count,
            ];
        }

        $result = [
            'courses'       => $data,
            'total_courses' => count( $data ),
        ];

        set_transient( $cache_key, $result, 5 * MINUTE_IN_SECONDS );

        return new WP_REST_Response( $result, 200 );
    }

    /**
     * Get paginated user enrollment data for a single course
     *
     * @param int             $course_id
     * @param WP_REST_Request $request
     * @return WP_REST_Response
     */
    private function get_course_enrollments( $course_id, $request ) {
        $course = get_post( $course_id );
        if ( ! $course || $course->post_type !== 'sfwd-courses' || $course->post_status !== 'publish' ) {
            return new WP_Error(
                'course_not_found',
                'Course not found.',
                [ 'status' => 404 ]
            );
        }

        $page     = max( 1, (int) $request->get_param( 'page' ) );
        $per_page = min( 250, max( 1, (int) $request->get_param( 'per_page' ) ) );

        // Check transient cache
        $cache_key = "qimah_analytics_enrollment_{$course_id}_p{$page}_pp{$per_page}";
        $cached    = get_transient( $cache_key );
        if ( false !== $cached ) {
            return new WP_REST_Response( $cached, 200 );
        }

        $users_query = learndash_get_users_for_course( $course_id, [], false );
        $all_user_ids = $users_query instanceof WP_User_Query ? $users_query->get_results() : [];
        $total_users = count( $all_user_ids );
        $total_pages = (int) ceil( $total_users / $per_page );

        // Count completions across ALL enrolled users (not just this page)
        // Prime meta cache for all users first
        update_meta_cache( 'user', $all_user_ids );

        $completed_count = 0;
        foreach ( $all_user_ids as $uid ) {
            $progress = learndash_course_progress( [
                'user_id'   => $uid,
                'course_id' => $course_id,
                'array'     => true,
            ] );
            if ( ! empty( $progress['percentage'] ) && (int) $progress['percentage'] === 100 ) {
                $completed_count++;
            }
        }

        // Paginate for user detail
        $offset   = ( $page - 1 ) * $per_page;
        $page_ids = array_slice( $all_user_ids, $offset, $per_page );

        $users_data = [];
        foreach ( $page_ids as $uid ) {
            $progress = learndash_course_progress( [
                'user_id'   => $uid,
                'course_id' => $course_id,
                'array'     => true,
            ] );
            $pct = isset( $progress['percentage'] ) ? (int) round( $progress['percentage'] ) : 0;
            $completed = $pct === 100;

            // Enrollment date from LD user meta
            $access_from = get_user_meta( $uid, 'course_' . $course_id . '_access_from', true );
            $enrolled_at = $access_from
                ? date( 'c', (int) $access_from )
                : null;

            // Last active from qimah-sec step visits
            $last_active = null;
            if ( class_exists( 'Qimah_Step_Visits' ) ) {
                $la = Qimah_Step_Visits::get_course_last_active( $uid, $course_id );
                if ( $la ) {
                    $last_active = date( 'c', strtotime( $la ) );
                }
            }

            $users_data[] = [
                'user_id'     => (int) $uid,
                'progress'    => $pct,
                'completed'   => $completed,
                'enrolled_at' => $enrolled_at,
                'last_active' => $last_active,
            ];
        }

        $result = [
            'course_id'       => $course_id,
            'title'           => $course->post_title,
            'enrolled_count'  => $total_users,
            'completed_count' => $completed_count,
            'users'           => $users_data,
            'page'            => $page,
            'total_pages'     => $total_pages,
            'total_users'     => $total_users,
        ];

        set_transient( $cache_key, $result, 5 * MINUTE_IN_SECONDS );

        return new WP_REST_Response( $result, 200 );
    }

    // =========================================================================
    // FEEDBACK
    // =========================================================================

    /**
     * GET /analytics/feedback
     *
     * Returns aggregated feedback stats per course.
     * Reuses Qimah_CEP_Feedback::get_course_feedback_stats().
     *
     * @param WP_REST_Request $request
     * @return WP_REST_Response
     */
    public function get_feedback( $request ) {
        if ( ! class_exists( 'Qimah_CEP_Feedback' ) ) {
            return new WP_REST_Response( [
                'courses' => [],
            ], 200 );
        }

        $course_id = $request->get_param( 'course_id' );

        if ( $course_id ) {
            $course = get_post( $course_id );
            if ( ! $course || $course->post_type !== 'sfwd-courses' || $course->post_status !== 'publish' ) {
                return new WP_Error(
                    'course_not_found',
                    'Course not found.',
                    [ 'status' => 404 ]
                );
            }
            $course_ids = [ $course_id ];
        } else {
            $course_ids = get_posts( [
                'post_type'   => 'sfwd-courses',
                'post_status' => 'publish',
                'numberposts' => -1,
                'fields'      => 'ids',
            ] );
        }

        $courses = [];
        foreach ( $course_ids as $cid ) {
            // Check transient cache
            $cache_key = "qimah_analytics_feedback_{$cid}";
            $cached    = get_transient( $cache_key );

            if ( false !== $cached ) {
                $courses[] = $cached;
                continue;
            }

            $stats = Qimah_CEP_Feedback::get_course_feedback_stats( (int) $cid );

            // Map recent_issues from CEP format to API format
            $recent_issues = [];
            if ( ! empty( $stats['recent_issues'] ) ) {
                foreach ( $stats['recent_issues'] as $issue ) {
                    $recent_issues[] = [
                        'topic_title' => $issue['topic_title'] ?? '',
                        'comment'     => $issue['comment_content'] ?? '',
                        'date'        => ! empty( $issue['comment_date'] )
                            ? date( 'c', strtotime( $issue['comment_date'] ) )
                            : null,
                    ];
                }
            }

            $entry = [
                'course_id'     => (int) $cid,
                'title'         => get_the_title( $cid ),
                'total'         => (int) ( $stats['total'] ?? 0 ),
                'up'            => (int) ( $stats['up'] ?? 0 ),
                'down'          => (int) ( $stats['down'] ?? 0 ),
                'percentage'    => (int) ( $stats['percentage'] ?? 0 ),
                'recent_issues' => $recent_issues,
            ];

            set_transient( $cache_key, $entry, 10 * MINUTE_IN_SECONDS );
            $courses[] = $entry;
        }

        return new WP_REST_Response( [ 'courses' => $courses ], 200 );
    }
}
```

**Key implementation notes for the implementer:**

- **`learndash_get_users_for_course()`** returns `WP_User_Query` - call `->get_results()` to get user ID array
- **`learndash_course_progress()`** returns `['percentage' => float, ...]` when `array => true`
- **`Qimah_Step_Visits::get_course_last_active()`** returns MySQL datetime string (not timestamp) - convert with `strtotime()`
- **`Qimah_CEP_Feedback::get_course_feedback_stats()`** returns `['total', 'up', 'down', 'percentage', 'recent_issues']` where `recent_issues` has keys `comment_content`, `comment_date`, `topic_title` (ARRAY_A from `$wpdb->get_results`). Does NOT return `comment_post_ID` or `topic_id` - the spec's `topic_id` field is dropped from the API response.
- **`completed_count` in `get_all_courses_summary()`** iterates all users - acceptable for ~8 courses / ~838 users. n8n calls this at most every few minutes.
- **`update_meta_cache('user', $page_ids)`** primes WP's meta cache for the page of users, avoiding N+1 queries on `get_user_meta()`
- **Transient keys include per_page** (`_pp{$per_page}`) since different page sizes produce different results. Spec says `_p{page}` only - this is an intentional improvement to prevent stale data.
- **`completed_count` is computed across ALL users**, not just the page slice. The loop iterates all enrolled users for completion counting, then paginates for user detail. Acceptable at ~838 enrolled users.

- [ ] **Step 2: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-rest-analytics.php
git commit -m "feat(api): add analytics REST controller

GET /analytics/enrollments - course enrollment + progress data
GET /analytics/feedback - aggregated feedback stats per course

Authenticated via read:analytics scope. Transient-cached."
```

---

### Task 4: Wire the new file + version bump

**Files:**
- Modify: `qimah-profile/qimah-profile.php`

- [ ] **Step 1: Add require_once for the new analytics class**

In `qimah-profile.php`, insert after the `class-qimah-rest-instructor.php` require (line 188):

```php
require_once QIMAH_PROFILE_DIR . 'includes/api/class-qimah-rest-analytics.php';
```

- [ ] **Step 2: Ensure the singleton is instantiated**

Check how existing API classes are instantiated. Looking at the codebase: singletons are loaded on-demand via hooks (`rest_api_init`). The constructor calls `add_action('rest_api_init', ...)` and `get_instance()` must be called once. Find where other singletons are initialized.

Search for `get_instance()` calls in `qimah-profile.php` to find the initialization block. Add:

```php
Qimah_REST_Analytics::get_instance();
```

alongside the existing `Qimah_REST_Calendar::get_instance()` and similar calls.

- [ ] **Step 3: Version bump (6.22.1 -> 6.23.0, minor - new feature)**

Update all 3 locations in `qimah-profile/qimah-profile.php`:

1. Plugin header: `* Version: 6.23.0`
2. Global constant: `define('QIMAH_PROFILE_VERSION', '6.23.0');`
3. Class constant: `const VERSION = '6.23.0';`

**Note:** This is a minor bump (new API endpoints = new feature), not patch.

**Important:** If this PR is stacked on the webhook wiring PR (#235), the current version will be `6.22.1`. If #235 hasn't merged yet, bump from whatever version is on main.

- [ ] **Step 4: Commit**

```bash
git add qimah-profile/qimah-profile.php
git commit -m "chore(profile): wire analytics API + bump to 6.23.0"
```

---

## Verification Checklist

After all tasks complete:

- [ ] **Scope exists:** `read:analytics` appears in both `SCOPES` and `RATE_LIMITS` constants
- [ ] **Webhook fires:** `user.registered` is in EVENTS const, has `add_action('user_register', ...)` in constructor, has `on_user_registered()` handler
- [ ] **Routes registered:** Both `/analytics/enrollments` and `/analytics/feedback` routes exist
- [ ] **Auth works:** Endpoints return 401 without API key, 403 with wrong scope, 200 with `read:analytics` scope
- [ ] **LearnDash guard:** `/analytics/enrollments` returns `{"courses":[],"error":"learndash_not_active"}` when LD is off
- [ ] **CEP guard:** `/analytics/feedback` returns `{"courses":[]}` when CEP is off (no error field - matches spec)
- [ ] **Pagination:** `/analytics/enrollments?course_id=X&per_page=5&page=2` returns correct slice
- [ ] **`per_page` capped at 250:** Values above 250 are clamped
- [ ] **Caching:** Second request within 5min (enrollments) / 10min (feedback) hits transient
- [ ] **Version bumped in all 3 locations**
- [ ] **`require_once` added** for new file
- [ ] **`get_instance()` called** so routes actually register
- [ ] **No `wp_` prefix hardcoded** anywhere in new code
