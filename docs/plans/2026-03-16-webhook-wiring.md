# Webhook Wiring Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire 4 webhook events (`session.started`, `commerce.order_paid`, `commerce.order_refunded`, `commerce.coupon_applied`) so the Qimah Eyes pipeline receives live data.

**Architecture:** All changes are in one file - `class-qimah-webhooks.php`. Add 4 `add_action()` calls in the constructor and 4 corresponding handler methods. Each handler calls `$this->dispatch()` with the event name and payload. Version bump in 3 locations.

**Tech Stack:** PHP, WordPress hooks, WooCommerce API

**Spec:** `docs/superpowers/specs/2026-03-15-webhook-wiring-design.md`

---

## File Structure

No new files. Two existing files modified:

| File | Responsibility | Change |
|------|---------------|--------|
| `qimah-profile/includes/api/class-qimah-webhooks.php` | Webhook dispatch + event handlers | Add 4 hook registrations in constructor + 4 handler methods |
| `qimah-profile/qimah-profile.php` | Plugin bootstrap + version | Patch version bump (3 locations) |

---

## Chunk 1: Hook Registration + Handler Methods

### Task 1: Register the 4 new hooks in the constructor

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php:170-183` (constructor, after WooCommerce native events block)

- [ ] **Step 1: Add hook registrations**

Insert after the existing `woocommerce_order_status_processing` line (line 173) and before the `// CEP feedback events` comment (line 175). Add a blank line separator and a comment block:

```php
        // Qimah Eyes pipeline events (session + commerce)
        add_action( 'wp_login', [ $this, 'on_session_started' ], 10, 2 );
        add_action( 'woocommerce_order_status_completed', [ $this, 'on_order_paid' ], 10, 2 );
        add_action( 'woocommerce_order_refunded', [ $this, 'on_order_refunded' ], 10, 2 );
        add_action( 'woocommerce_applied_coupon', [ $this, 'on_coupon_applied' ], 10, 1 );
```

**Why `on_order_paid` hooks the same WC action as `on_woocommerce_order_completed`:** Both fire on `woocommerce_order_status_completed`. The existing handler dispatches `woocommerce.order_completed` (raw WC format via `format_wc_order()`). The new handler dispatches `commerce.order_paid` (pipeline-specific format with moyasar_channel, structured items, coupons). Both are intentional - the pipeline subscribes to `commerce.order_paid`, existing integrations use `woocommerce.order_completed`.

- [ ] **Step 2: Verify constructor compiles**

Visual check: count `add_action` calls. Should now be 22 total (18 existing + 4 new). No duplicate method names. All methods referenced will be created in Tasks 2-5.

- [ ] **Step 3: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): register 4 Qimah Eyes pipeline hooks

Hook wp_login, woocommerce_order_status_completed (commerce.order_paid),
woocommerce_order_refunded, and woocommerce_applied_coupon in constructor."
```

---

### Task 2: Add `on_session_started` handler

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php` (insert after `on_woocommerce_order_processing` method, ~line 642, before `format_wc_order`)

- [ ] **Step 1: Add the handler method**

Insert before the `format_wc_order` method (line 644). Place it with the other WC/session handlers:

```php
    /**
     * Handle session started event (Qimah Eyes pipeline)
     *
     * @param string  $user_login Username
     * @param WP_User $user       User object
     */
    public function on_session_started( $user_login, $user ) {
        $this->dispatch( 'session.started', [
            'user_id'    => $user->ID,
            'ip_address' => isset( $_SERVER['REMOTE_ADDR'] ) ? sanitize_text_field( wp_unslash( $_SERVER['REMOTE_ADDR'] ) ) : '',
            'user_agent' => isset( $_SERVER['HTTP_USER_AGENT'] ) ? sanitize_text_field( wp_unslash( $_SERVER['HTTP_USER_AGENT'] ) ) : '',
            'timestamp'  => current_time( 'c' ),
        ] );
    }
```

**Key notes for implementer:**
- `wp_login` passes `$user_login` (string) and `$user` (WP_User object) - both are needed in the signature even though we only use `$user`
- Do NOT include session tokens in the payload - sensitive data
- `current_time('c')` produces ISO 8601 with timezone offset (e.g. `2026-03-15T10:00:00+03:00`) - matches existing handler convention

- [ ] **Step 2: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): add on_session_started handler

Dispatches session.started with user_id, IP, user agent, and timestamp.
Hooks wp_login for Qimah Eyes pipeline live session ingestion."
```

---

### Task 3: Add `on_order_paid` handler

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php` (insert after `on_session_started`)

- [ ] **Step 1: Add the handler method**

Insert immediately after `on_session_started`:

```php
    /**
     * Handle order paid event (Qimah Eyes pipeline)
     *
     * Fires on woocommerce_order_status_completed alongside on_woocommerce_order_completed.
     * This handler dispatches commerce.order_paid with a pipeline-specific payload (moyasar_channel,
     * structured items, coupons). The existing handler dispatches woocommerce.order_completed
     * with raw WC format. Both are intentional.
     *
     * @param int           $order_id Order ID
     * @param WC_Order|null $order    Order object (may be null if caller passes only ID)
     */
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

**Key notes for implementer:**
- `$order` param defaults to `null` because QCX sometimes fires `do_action('woocommerce_order_status_completed', $order_id)` with only the ID
- Use `instanceof WC_Order` check (not `!$order`) because `$order_id` is truthy but not an order object
- `_qimah_moyasar_channel` meta is set by `mu-plugins/qimah-moyasar-channel.php` (Mada/Visa/MC/Apple Pay)
- `get_coupon_codes()` returns array of coupon code strings

- [ ] **Step 2: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): add on_order_paid handler

Dispatches commerce.order_paid with structured items, moyasar_channel,
coupons. Handles $order being null when QCX replays WC hooks with ID only."
```

---

### Task 4: Add `on_order_refunded` handler

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php` (insert after `on_order_paid`)

- [ ] **Step 1: Add the handler method**

Insert immediately after `on_order_paid`:

```php
    /**
     * Handle order refunded event (Qimah Eyes pipeline)
     *
     * Uses woocommerce_order_refunded (not status transition hook) because it passes
     * the $refund_id directly, giving us the exact refund object with amount and reason.
     *
     * @param int $order_id  Order ID
     * @param int $refund_id Refund ID
     */
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

**Key notes for implementer:**
- `woocommerce_order_refunded` fires on BOTH full and partial refunds
- `$refund->get_amount()` can be negative in WC internals - `abs()` normalizes it
- `partial` is `true` when order status is NOT `refunded` (i.e. still `completed` after a partial refund)

- [ ] **Step 2: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): add on_order_refunded handler

Dispatches commerce.order_refunded with refund_amount, partial flag, reason.
Uses woocommerce_order_refunded hook for exact refund object access."
```

---

### Task 5: Add `on_coupon_applied` handler

**Files:**
- Modify: `qimah-profile/includes/api/class-qimah-webhooks.php` (insert after `on_order_refunded`)

- [ ] **Step 1: Add the handler method**

Insert immediately after `on_order_refunded`:

```php
    /**
     * Handle coupon applied event (Qimah Eyes pipeline)
     *
     * Hooks WC's native woocommerce_applied_coupon directly. QCX_Cart_Manager only handles
     * Tabby coupon enforcement (remove/restore), not normal coupon application.
     *
     * @param string $coupon_code Coupon code
     */
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

**Key notes for implementer:**
- `WC()->cart` guard is needed - this hook can theoretically fire outside cart context
- `get_current_user_id()` is correct here (user is applying coupon in their own session, not a hook that clears the user)
- `get_discount_type()` returns strings like `percent`, `fixed_cart`, `fixed_product`

- [ ] **Step 2: Commit**

```bash
git add qimah-profile/includes/api/class-qimah-webhooks.php
git commit -m "feat(webhooks): add on_coupon_applied handler

Dispatches commerce.coupon_applied with discount_type, amount, cart_total.
Hooks WC native woocommerce_applied_coupon, bypasses QCX entirely."
```

---

## Chunk 2: Version Bump

### Task 6: Bump qimah-profile version

**Files:**
- Modify: `qimah-profile/qimah-profile.php` (3 locations)

Current version: `6.22.0`. New version: `6.22.1` (patch - wiring existing event definitions, no new user-facing features).

- [ ] **Step 1: Update plugin header**

In `qimah-profile/qimah-profile.php`, find:
```
 * Version: 6.22.0
```
Replace with:
```
 * Version: 6.22.1
```

- [ ] **Step 2: Update global constant**

In `qimah-profile/qimah-profile.php`, find:
```php
define('QIMAH_PROFILE_VERSION', '6.22.0');
```
Replace with:
```php
define('QIMAH_PROFILE_VERSION', '6.22.1');
```

- [ ] **Step 3: Update class constant**

In `qimah-profile/qimah-profile.php`, find:
```php
    const VERSION = '6.22.0';
```
Replace with:
```php
    const VERSION = '6.22.1';
```

- [ ] **Step 4: Commit**

```bash
git add qimah-profile/qimah-profile.php
git commit -m "chore(profile): bump version to 6.22.1"
```

---

## Verification Checklist

After all tasks are complete, verify:

- [ ] **No duplicate hooks:** `on_order_paid` and `on_woocommerce_order_completed` both hook `woocommerce_order_status_completed` - this is intentional (two different webhook events dispatched). Confirm both are present.
- [ ] **Method names unique:** No existing method named `on_session_started`, `on_order_paid`, `on_order_refunded`, or `on_coupon_applied`.
- [ ] **Event names match EVENTS const:** `session.started` (line 61), `commerce.order_paid` (line 85), `commerce.order_refunded` (line 86), `commerce.coupon_applied` (line 93) - all already defined.
- [ ] **Version bumped in all 3 locations:** header, `QIMAH_PROFILE_VERSION`, `const VERSION`.
- [ ] **No booking hooks added:** Booking events are already wired via `fire_webhook_event()` - do NOT add listeners.
- [ ] **PHP syntax check:** `php -l qimah-profile/includes/api/class-qimah-webhooks.php` (will fail on this dev machine without WP, but check for obvious syntax errors in the diff).
