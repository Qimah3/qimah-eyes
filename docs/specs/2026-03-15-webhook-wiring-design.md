# Webhook Wiring for Qimah Eyes Pipeline - Design Spec

**Date:** 2026-03-15
**Plugin:** qimah-profile (single-plugin change)
**Status:** Draft
**Related:** `2026-03-15-qimah-eyes-data-pipeline-design.md` (pipeline spec), `2026-03-11-analytics-api-design.md` (API pull endpoints)

---

## Summary

Wire 4 webhook events in qimah-profile that the Qimah Eyes pipeline depends on for live data ingestion. Three events are defined in `Qimah_Webhooks::EVENTS` but have no handler (`session.started`, `commerce.order_paid`, `commerce.order_refunded`). The fourth (`commerce.coupon_applied`) hooks WC's native coupon action directly.

**Note:** Booking events (`booking.created/confirmed/completed/cancelled/no_show`) are NOT dead - `Qimah_Instructor_Booking_Service::fire_webhook_event()` already calls `Qimah_Webhooks::trigger()` directly (line ~1014). They fire correctly today and need no changes.

This spec covers webhook wiring only. For the API pull endpoints (`GET /analytics/enrollments`, `GET /analytics/feedback`, `read:analytics` scope, `user.registered` webhook), see the analytics API spec.

---

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Commerce order events | Hook WC directly (`woocommerce_order_status_completed/refunded`) instead of waiting for QCX to emit `qcx_order_paid` | QCX never emits these events; direct WC hooks are simpler and have no cross-plugin dependency |
| Commerce coupon event | Hook WC's native `woocommerce_applied_coupon` directly in webhooks class | QCX_Cart_Manager only handles Tabby coupon enforcement (remove/restore), not normal coupon application. WC core's `apply_coupon()` fires this hook for all coupons. |
| Cart abandoned | Deferred | Needs a detection mechanism (cron-based cart scanning); not just wiring - a feature unto itself |
| Booking events | Already wired - no changes needed | `fire_webhook_event()` in booking service already calls `Qimah_Webhooks::trigger()` directly |
| session.started | Hook `wp_login` | Standard WP hook, fires on every login |
| Refund hook | `woocommerce_order_refunded` (not status transition hook) | Passes `$refund_id` so we get the exact refund object, not a guess from `get_refunds()[0]` |
| Dispatch method | Use `$this->dispatch()` not `$this->trigger()` | `trigger()` is static; instance handlers use `dispatch()` per existing patterns |
| Sharing events | Deferred | qimah-sec owns sharing detection but has no webhook infrastructure; different scope/risk |
| user.registered | Not in this spec | Already covered by analytics API spec |

---

## Events to Wire

### 1. session.started

**WP Hook:** `wp_login` (priority 10, 2 args: `$user_login`, `$user`)
**Write target:** Pipeline's Activity webhook workflow

```php
public function on_session_started( $user_login, $user ) {
    $this->dispatch( 'session.started', [
        'user_id'    => $user->ID,
        'ip_address' => isset( $_SERVER['REMOTE_ADDR'] ) ? sanitize_text_field( wp_unslash( $_SERVER['REMOTE_ADDR'] ) ) : '',
        'user_agent' => isset( $_SERVER['HTTP_USER_AGENT'] ) ? sanitize_text_field( wp_unslash( $_SERVER['HTTP_USER_AGENT'] ) ) : '',
        'timestamp'  => current_time( 'c' ),
    ] );
}
```

**Note:** Don't include raw session token in payload - it's sensitive. IP + user agent is sufficient for the pipeline's `daily_activity` and `session_patterns` tables.

**Timezone:** All timestamps use `current_time('c')` (WP local time with offset, e.g. `2026-03-15T10:00:00+03:00`). The pipeline normalizes to UTC on ingestion. This matches the convention used by existing webhook handlers.

---

### 2. commerce.order_paid

**WP Hook:** `woocommerce_order_status_completed` (priority 10, 2 args: `$order_id`, `$order`)
**Write target:** Pipeline's Revenue webhook workflow -> `orders` table

```php
public function on_order_paid( $order_id, $order = null ) {
    if ( ! $order instanceof WC_Order ) {
        $order = wc_get_order( $order_id );
    }
    if ( ! $order ) {
        return;
    }

    $items = [];
    foreach ( $order->get_items() as $item ) {
        $items[] = [
            'product_id'   => $item->get_product_id(),
            'product_name' => $item->get_name(),
            'quantity'     => $item->get_quantity(),
            'total'        => (float) $item->get_total(),
        ];
    }

    $this->dispatch( 'commerce.order_paid', [
        'order_id'        => $order_id,
        'user_id'         => $order->get_customer_id(),
        'items'           => $items,
        'total'           => (float) $order->get_total(),
        'currency'        => $order->get_currency(),
        'payment_method'  => $order->get_payment_method(),
        'moyasar_channel' => $order->get_meta( '_qimah_moyasar_channel' ) ?: null,
        'coupons'         => $order->get_coupon_codes(),
        'timestamp'       => current_time( 'c' ),
    ] );
}
```

**Relationship to existing `woocommerce.order_completed`:** Both fire on the same WC hook. The existing `woocommerce.order_completed` handler stays as-is (raw WC event for backward compatibility). `commerce.order_paid` is the semantic event the pipeline subscribes to, with a richer payload (moyasar_channel, coupons, structured items).

**Integration note:** `$order` is defaulted to `null` because parts of QCX manually fire `do_action('woocommerce_order_status_completed', $order_id)` with only the order ID. The handler must always be safe with just `$order_id`.

**Note on "processing" orders:** Moyasar orders transition through `processing` -> `completed`. We fire `commerce.order_paid` only on `completed` (not `processing`) because that's when payment is confirmed and course access is granted. The existing `woocommerce.order_processing` event already captures the processing stage if the pipeline needs it later.

---

### 3. commerce.order_refunded

**WP Hook:** `woocommerce_order_refunded` (priority 10, 2 args: `$order_id`, `$refund_id`)
**Write target:** Pipeline's Revenue webhook workflow -> `orders` table (status update)

This is the refund-specific hook (not the status transition hook `woocommerce_order_status_refunded`). It fires on every refund (full or partial) and passes the `$refund_id`, giving us the exact refund object with its amount and reason - no guessing from `get_refunds()[0]`.

```php
public function on_order_refunded( $order_id, $refund_id ) {
    $order  = wc_get_order( $order_id );
    $refund = wc_get_order( $refund_id );
    if ( ! $order || ! $refund ) {
        return;
    }

    $this->dispatch( 'commerce.order_refunded', [
        'order_id'      => $order_id,
        'refund_id'     => $refund_id,
        'user_id'       => $order->get_customer_id(),
        'refund_amount' => abs( (float) $refund->get_amount() ),
        'total'         => (float) $order->get_total(),
        'partial'       => $order->get_status() !== 'refunded',
        'reason'        => $refund->get_reason(),
        'timestamp'     => current_time( 'c' ),
    ] );
}
```

---

### 4. commerce.coupon_applied

**WP Hook:** `woocommerce_applied_coupon` (priority 10, 1 arg: `$coupon_code`)
**Write target:** Pipeline's Revenue webhook workflow -> `coupons` table

**Why not QCX?** `QCX_Cart_Manager` does not own normal coupon application - it only removes/restores coupons during Tabby payment enforcement (`class-qcx-cart-manager.php:218,257`). Ordinary coupon applications go through WC core's `WC_Cart::apply_coupon()`, which fires `woocommerce_applied_coupon`. Hooking this directly in the webhooks class catches all coupons regardless of source.

```php
public function on_coupon_applied( $coupon_code ) {
    $cart = WC()->cart;
    if ( ! $cart ) {
        return;
    }

    $coupon = new WC_Coupon( $coupon_code );

    $this->dispatch( 'commerce.coupon_applied', [
        'coupon_code'   => $coupon_code,
        'user_id'       => get_current_user_id(),
        'discount_type' => $coupon->get_discount_type(),
        'amount'        => (float) $coupon->get_amount(),
        'cart_total'    => (float) $cart->get_cart_contents_total(),
        'timestamp'     => current_time( 'c' ),
    ] );
}
```

**Commerce Bridge:** Not involved. This hooks WC directly in the webhooks class, bypassing the `qcx_coupon_applied` -> bridge path entirely. The bridge's coupon handler remains for any future QCX-specific coupon events but is not triggered by this flow.

---

### 5-9. booking.* - Already Wired (No Changes Needed)

`Qimah_Instructor_Booking_Service::fire_webhook_event()` (line ~1010) already calls `Qimah_Webhooks::trigger()` directly for all booking lifecycle events (`booking.created`, `booking.confirmed`, `booking.completed`, `booking.cancelled`, `booking.no_show`). The payloads include booking fields plus n8n helper data (calendar event details, email context).

**Do NOT add `add_action('qimah_booking_*')` listeners in the webhooks class** - this would cause every booking event to dispatch twice (once from `fire_webhook_event()` and once from the new listener).

---

## Hook Registration

New `add_action()` calls in `Qimah_Webhooks::__construct()`, alongside existing hook listeners:

```php
// Session
add_action( 'wp_login', [ $this, 'on_session_started' ], 10, 2 );

// Commerce (direct WC hooks)
add_action( 'woocommerce_order_status_completed', [ $this, 'on_order_paid' ], 10, 2 );  // $order may be null if caller passes only ID
add_action( 'woocommerce_order_refunded', [ $this, 'on_order_refunded' ], 10, 2 );       // passes $refund_id, fires on full + partial
add_action( 'woocommerce_applied_coupon', [ $this, 'on_coupon_applied' ], 10, 1 );
```

**Not registered here:** Booking events (already dispatched by booking service via `fire_webhook_event()`).

---

## File Changes

| File | Change | Lines (est.) |
|------|--------|-------------|
| `qimah-profile/includes/api/class-qimah-webhooks.php` | Add 4 `add_action()` calls in constructor + 4 handler methods (`on_session_started`, `on_order_paid`, `on_order_refunded`, `on_coupon_applied`) | +80 |
| `qimah-profile/qimah-profile.php` | Version bump (patch) | 3 locations |

**QCX no longer modified.** Coupon tracking now hooks WC's native `woocommerce_applied_coupon` directly in the webhooks class, bypassing QCX entirely.

---

## Interaction with Existing Events

| New Event | Existing Similar Event | Coexistence |
|-----------|----------------------|-------------|
| `commerce.order_paid` | `woocommerce.order_completed` | Both fire on same WC hook - each completed order triggers two webhook deliveries (two Action Scheduler tasks). Pipeline subscribes to `commerce.order_paid` (richer payload). Existing `woocommerce.order_completed` stays for backward compatibility. Acceptable at current volume (~838 enrolled users). |
| `commerce.order_refunded` | (none) | New - no existing refund webhook. |
| `commerce.coupon_applied` | (none) | New - hooks WC directly (`woocommerce_applied_coupon`), does not use Commerce Bridge. |
| `session.started` | (none) | New - complements existing `session.terminated`. |
| `booking.*` | (already dispatched) | Already wired via `fire_webhook_event()` in booking service. No changes needed. |

---

## Out of Scope

- `sharing.*` events (qimah-sec - different plugin, no webhook infrastructure)
- `commerce.cart_abandoned` (needs detection mechanism, not just wiring)
- `user.registered` (covered by analytics API spec)
- `booking.reminder`, `booking.meeting_url_set` (not needed by pipeline schema)
- `commerce.subscription_*` (no subscriptions in production)
- `course.started` (derivable from first `topic.completed`)
- `user.banned` / `user.unbanned` (low volume, low priority)
- `discord.roles_synced` (low value for analytics)
- `points.milestone` (derivable from `points.awarded`)

---

## Verification

After deployment, verify each event fires correctly:

1. **session.started:** Log in to WP -> check webhook delivery log in Admin Hub
2. **commerce.order_paid:** Complete a test order -> verify webhook fires with moyasar_channel and items
3. **commerce.order_refunded:** Refund a test order -> verify webhook fires with refund_amount
4. **commerce.coupon_applied:** Apply a coupon in cart -> verify QCX emits and Commerce Bridge dispatches
5. **booking.*:** Create/confirm/complete/cancel a booking -> verify each event fires with correct payload

Check n8n webhook reception logs to confirm the pipeline receives and processes each event type.
