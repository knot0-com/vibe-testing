# Example Vibe Test: E-Commerce Checkout

This is a complete example of a vibe test case. Adapt the structure to your own system.

---

## VT-1: First-Time Buyer With Payment Failure

### Persona

**Sarah** — First-time customer. Shopping on her phone during a commute. Expects checkout to take under 60 seconds. Low patience for errors. Has never created an account on this platform.

### Environment

- **Client:** Mobile browser (iOS Safari, 3G connection)
- **Backend:** Microservices — auth, payments, inventory, orders, notifications, shipping
- **Scale:** Normal weekday traffic
- **Payment:** Visa credit card (first attempt will be declined by issuer)

### Goal

> "I want to buy these 3 items, pay with my credit card, and get a confirmation email within a minute."

### Scenario

#### Step 1: Sarah adds items to cart and starts checkout

She browses the catalog, adds 3 items, and taps "Checkout." The system creates a cart and begins the checkout flow.

**Primitives:**
- `cart-spec.md`: Cart creation, item validation, price calculation
- `inventory-spec.md`: Stock availability check, stock reservation/hold
- `auth-spec.md`: Guest checkout vs. account creation

**Questions:**
- Q1.1: Does the inventory spec reserve stock when items are added to cart, or only at checkout start? If at cart-add time, what's the hold duration?
- Q1.2: The auth spec defines JWT tokens with a 15-minute TTL. The checkout flow could take longer on a slow connection. What happens if the token expires mid-checkout?
- Q1.3: Can Sarah check out as a guest, or must she create an account? The auth spec defines account creation but the checkout spec may assume an authenticated user.

#### Step 2: Sarah enters shipping details

She fills in her address. The system validates it and calculates shipping options.

**Primitives:**
- `shipping-spec.md`: Address validation, rate calculation, delivery estimates
- `orders-spec.md`: Order creation (draft state)
- `tax-spec.md`: Tax calculation based on shipping destination

**Questions:**
- Q2.1: The shipping spec calls an external address validation API. What happens if that API times out on Sarah's slow 3G connection? Is there a fallback?
- Q2.2: At what point is the order record created — when Sarah enters shipping details, or after payment succeeds? The orders spec should define the initial state.
- Q2.3: Tax calculation depends on the shipping address. If Sarah changes her address after seeing the total, does the order spec define a recalculation flow?

#### Step 3: Payment is declined, Sarah retries with a different card

Sarah enters her Visa. The payment processor declines it (insufficient funds). She enters a different card.

**Primitives:**
- `payments-spec.md`: Payment attempt, decline handling, retry policy, idempotency
- `inventory-spec.md`: Stock hold duration during payment retry
- `orders-spec.md`: Order state transitions on payment failure

**Questions:**
- Q3.1: The payments spec says "retry up to 3 times with exponential backoff." But Sarah isn't retrying the same card — she's entering a new one. Is a new card a "retry" or a new payment attempt? Does the idempotency key reset?
- Q3.2: The inventory spec holds stock for 5 minutes. If the decline + new card entry + second attempt takes 6 minutes, is the stock released? Can another customer buy the same item while Sarah is retrying?
- Q3.3: What order state is Sarah's order in during the retry? The orders spec defines `pending_payment` and `failed`, but not `retrying_payment`. Which transition applies?

#### Step 4: Second payment succeeds, order is confirmed

The second card is charged. The order transitions to confirmed. Inventory is decremented.

**Primitives:**
- `payments-spec.md`: Successful charge, payment confirmation
- `orders-spec.md`: State transition to `confirmed`
- `inventory-spec.md`: Stock hold → permanent decrement
- `notifications-spec.md`: Confirmation email trigger

**Questions:**
- Q4.1: The payment succeeded but the order confirmation fails (e.g., the orders service is briefly unavailable). The customer is charged but has no order. Does the payments spec define a rollback? Does the orders spec define eventual consistency?
- Q4.2: The notifications spec says "send confirmation email on order.confirmed event." What is the delivery SLA? If the email service is down, does it retry? Is there a dead-letter queue?
- Q4.3: Sarah paid with her second card. Is the first (declined) attempt recorded anywhere? The payments spec should define how to handle multiple attempts per order for reconciliation.

#### Step 5: Sarah checks order status 10 minutes later

She opens the order status page to see if her order is being processed.

**Primitives:**
- `orders-spec.md`: Order status API, state machine
- `auth-spec.md`: Session persistence (did Sarah create an account, or is she a guest?)
- `shipping-spec.md`: Tracking number assignment

**Questions:**
- Q5.1: If Sarah checked out as a guest, how does she access her order status? By email link? Order number + email lookup? The auth spec and orders spec must agree on guest access.
- Q5.2: The shipping spec says a tracking number is assigned "when the shipment is created." But the order was confirmed 10 minutes ago. What state does Sarah see before a tracking number exists? Is there a `processing` state?
- Q5.3: If Sarah's session expired (15-minute JWT TTL), can she still view her order? The auth spec may block access to order data for expired sessions.

### Spec Coverage Matrix

| Spec Doc | Steps Hit | Coverage |
|----------|-----------|----------|
| `auth-spec.md` | 1,5 | JWT TTL defined; guest checkout path unclear; expired session + order access gap |
| `cart-spec.md` | 1 | Cart creation covered |
| `inventory-spec.md` | 1,3,4 | Stock hold covered; hold-vs-retry timing conflict |
| `orders-spec.md` | 2,3,4,5 | Core state machine covered; `retrying_payment` state missing; guest access gap |
| `payments-spec.md` | 3,4 | Retry policy covered; new-card-vs-retry ambiguity; multi-attempt reconciliation gap |
| `notifications-spec.md` | 4 | Email trigger covered; delivery failure handling gap |
| `shipping-spec.md` | 2,5 | Rate calculation covered; address API timeout gap; pre-tracking state gap |
| `tax-spec.md` | 2 | Tax calculation covered; recalculation on address change unclear |
| `admin-spec.md` | — | Not exercised |
| `analytics-spec.md` | — | Not exercised |
| `returns-spec.md` | — | Not exercised |

---

## Expected Output

After simulation, the gap report would look like:

### BLOCKING

| ID | Gap | Affected Steps | Recommended Fix |
|----|-----|---------------|-----------------|
| G-B1 | Payment retry duration can exceed inventory hold — stock may be sold to another customer mid-retry | 3 | Align timing: extend hold on active payment retry, or define hold-refresh mechanism in `inventory-spec.md` |
| G-B2 | Auth token TTL (15 min) shorter than possible checkout duration — session can expire mid-checkout | 1, 5 | Add token refresh flow or checkout-scoped session in `auth-spec.md` |
| G-B3 | Payment succeeds but order confirmation fails — customer is charged with no order | 4 | Define saga/compensation pattern across `payments-spec.md` and `orders-spec.md` |

### DEGRADED

| ID | Gap | Affected Steps | Workaround |
|----|-----|---------------|-----------|
| G-D1 | No `retrying_payment` order state — ambiguous state during card switch | 3 | Keep order in `pending_payment`; less precise but functional |
| G-D2 | Guest checkout order access undefined | 1, 5 | Send order link in confirmation email; no account required |
| G-D3 | No email delivery failure handling | 4 | Retry with exponential backoff; accept occasional missed emails |
| G-D4 | Address validation API timeout not handled | 2 | Skip validation on timeout; flag for manual review |

### COSMETIC

| ID | Gap | Affected Steps |
|----|-----|---------------|
| G-C1 | No pre-tracking order state description for customer-facing UI | 5 |
| G-C2 | Failed payment attempts not surfaced in admin order view | 3 |
