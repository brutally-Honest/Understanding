# Idempotency

Reference notes

---

## 1. What is it

An operation is **idempotent** if running it once or running it N times produces the same end state.

> Elevator button: pressing it 10 times doesn't send the elevator 10 times. It registers "go to floor 5" once.

Key distinction: idempotency is about **end state**, not about whether the operation *runs* multiple times. The operation can execute repeatedly — the side effect must not stack.

---

## 2. Why required

Core problem: **networks are unreliable, but retries are necessary for reliability.**

- Client sends `charge ₹500`.
- Server processes it, debits money.
- Response is lost on the way back (timeout / connection drop).
- Client has no idea if it worked → retries.

Without idempotency: retry = double charge.
With idempotency: retry = server recognizes "already did this," returns original result, no double charge.

**Idempotency is what makes retries safe.** Without it, you're stuck choosing between "don't retry" (bad for reliability) and "retry blindly" (bad for correctness).

Related concept: **exactly-once processing doesn't exist for free in distributed systems.** You natively only get at-least-once or at-most-once delivery. What you actually build is:

```
at-least-once delivery + idempotent processing  ≈  behaves like exactly-once
```

This framing is useful to state explicitly in system design discussions — exactly-once is an illusion constructed from retries + idempotency, not a primitive guarantee.

---

## 3. Idempotent HTTP methods

| Method | Idempotent? | Why |
|---|---|---|
| GET | Yes | Read-only, no state change |
| PUT | Yes (by spec) | `PUT /users/5 {name: "Karthik"}` run N times → same end state |
| DELETE | Yes (by spec) | User gone after 1st call, still gone after Nth call |
| POST | **No** | `POST /orders` creates a new resource every call — this is the dangerous one |
| PATCH | No (by spec, often made idempotent in practice) | Depends on semantics — `set status=SHIPPED` is idempotent, `increment count` is not |

**The entire idempotency engineering problem in practice is: how do you make POST (and other "create new thing" calls) safe to retry?** GET/PUT/DELETE get it for free from their semantics; POST does not, and most real systems' idempotency work is specifically about wrapping POST safely.

---

## 4. Real-world / company-specific examples

- **Stripe** — `Idempotency-Key` header on POST requests (e.g. payment creation). Response cached ~24h; retried request with same key returns the original response, no second charge.
- **Razorpay / PayPal** — order creation APIs require client-generated idempotency key / order ID to prevent duplicate payment on retry.
- **Kafka producers** — `enable.idempotence=true`. Each producer gets a sequence number per partition; broker dedupes retried sends caused by broker timeout, so retries don't create duplicate log entries.
- **AWS APIs** — e.g. EC2 `RunInstances` accepts a `ClientToken` so a retried "launch instance" call doesn't spin up multiple servers instead of one.
- **Database migrations** — `CREATE TABLE IF NOT EXISTS`, upserts (`INSERT ... ON CONFLICT DO UPDATE`) — re-running a migration script is safe by design.
- **Webhooks (Stripe, GitHub, Shopify)** — delivery is at-least-once by design. Receiving endpoint must dedupe using the event ID in the payload, because the same webhook can legitimately arrive twice.

---

## 5. Use case deep dive: Payments (Stripe PaymentIntent model)

Two separate calls, two separate jobs, at two different points in the flow.

### Call 1 — Create PaymentIntent ("I'm about to attempt a payment")

```
User clicks Buy
   → Frontend → Your Backend: "create payment intent for this cart"
   → Your Backend → Stripe: POST /payment_intents { amount }
   → Stripe creates object, status = requires_payment_method
   → Stripe → Your Backend → Frontend: client_secret
```

- No card touched yet. No money moved.
- Must happen on **your backend** — needs your secret API key, never exposed to browser.
- This call itself uses an idempotency key on your backend, so a network blip doesn't create two PaymentIntents for one cart.

### Call 2 — Confirm PaymentIntent ("here's the card, attempt the charge")

```
Frontend → Stripe directly: confirmCardPayment(client_secret, card details)
   → Stripe → Bank: actually attempt the charge
```

- Happens **browser-to-Stripe directly**, not through your backend — card details should never pass through your servers (PCI compliance).
- This is the call that can fail/retry (decline, network drop, 2FA timeout). Every retry references the **same PaymentIntent ID**, so Stripe treats it as another attempt at the same intent, not a new charge.

### Why split into two calls instead of one

1. Time gap — intent can be created before the user has even entered card details.
2. Retry safety — step 1 happens once; step 2 can retry any number of times against the same anchor ID without risk of double-charging.
3. Trust boundary separation — secret key stays server-side (step 1), card data stays browser-to-Stripe only (step 2).

### Dual notification (important, easy to miss)

Stripe confirms outcome through **two independent channels**:

1. `confirmCardPayment()` resolves in-browser → immediate UI feedback.
2. **Webhook** (`payment_intent.succeeded`) sent server-to-server, independent of the browser.

Why both: the browser-side signal can be lost (tab closed right after paying, crash, refresh) even though the charge succeeded. The webhook is the **durable source of truth** — your DB should be updated by the webhook, not by trusting the frontend's success callback.

```
                Your Backend ──creates──▶ Stripe (PaymentIntent, status: requires_payment_method)
                     │                         │
                     │◀──── client_secret ──────┘
                     ▼
                Frontend (has client_secret)
                     │
                     ▼
       Frontend ── confirmCardPayment ──▶ Stripe (direct, browser-to-Stripe)
                     │                         │
                     │◀── success/fail ────────┤
                     │  (shown to user)        ▼
                     │               Stripe ── webhook ──▶ Your Backend
                     │                                     (DB updated here,
                     │                                      independent of browser)
```

### Retry vs fresh intent — the actual decision rule

| Situation | What to do |
|---|---|
| Same in-flight checkout, network retried automatically | Reuse same idempotency key / PaymentIntent ID |
| Page refreshed mid-payment | Do NOT blindly generate new key. Check if a PaymentIntent already exists for this cart/session, fetch its live status first |
| User genuinely starts a new purchase (new cart/session) | New PaymentIntent, new key — this is correct, not a bug |
| Polling for current status after reconnect | This is a **read**, not a retried write — always check live state (Stripe / your DB via webhook), never serve a frozen cached response here |

**Important distinction:** idempotent retry of a *write* → return the cached/stored result of that exact write. Checking status of something async (post-refresh) → always re-check live state. Conflating these two causes a real bug class: showing "processing" forever when the payment actually succeeded in the background.

---

## 6. Use case: Event processing (Kafka / async consumers)

Different failure mode than HTTP, same underlying principle.

- Kafka delivery is generally **at-least-once** — a consumer can receive the same message more than once (rebalance, consumer crash before offset commit, producer retry).
- **Idempotent producer** (`enable.idempotence=true`): broker dedupes retried sends using producer ID + per-partition sequence number. Solves duplicate writes to the log itself.
- **Idempotent consumer pattern**: even with producer-side idempotence, consumers can still process the same message twice (e.g., crash after processing but before committing offset). Two ways to handle:
  1. Track processed message IDs (dedup table/cache) before applying side effects.
  2. Design the operation itself to be naturally idempotent — `set status = SHIPPED` is safe to apply twice; `increment shipped_count` is not.

Same elimination logic as the payments case: push idempotency to whichever layer is doing the side effect, because delivery guarantees alone are never enough.

---

## 7. Use case: Notifications (email / SMS / push)

Lower stakes than payments, but the same retry-causes-duplicates problem shows up constantly.

- A "send welcome email" job fails to get its success ack (timeout, worker crash after sending but before marking done) → job gets retried → user gets the same email twice.
- Worse version: a Kafka consumer (Section 6) processing an "order placed" event triggers a notification as a side effect. If the consumer re-processes the same event (at-least-once delivery) without idempotency, the user gets duplicate "your order shipped" emails/SMS for one shipment.

**Why it's lower priority than payments, but not ignorable:**
- No financial/data-integrity damage — worst case is user annoyance, brand trust erosion, or (for SMS) real cost per message at scale.
- But it's an easy, high-frequency failure mode because notification sends are almost always triggered as a *side effect* of some other event (order placed, payment succeeded, password reset requested) — and that triggering event is exactly the kind of thing that gets retried/redelivered.

**How it's handled in practice:**
- Idempotency key = a stable identifier tied to the *event*, not a random UUID per send attempt — e.g. `order_id + "shipped"` or `notification_type + user_id + event_id`. This way, "send shipped email for order #123" is naturally the same key no matter how many times the underlying event is redelivered.
- Dedup table: `(notification_key)` with a unique constraint — before sending, check/insert; if the row already exists, skip the send instead of re-sending.
- Services like SendGrid/Twilio/SES generally don't dedupe for you — the idempotency has to live in *your* layer (the thing deciding "should I call the send API right now"), not the provider's API itself (unlike Stripe, which dedupes server-side for you on `Idempotency-Key`).
- Password reset / OTP emails are a special case worth noting: here you sometimes *want* a retry to actually resend (user didn't receive it, clicks "resend"), so the idempotency window is deliberately short or scoped per-request rather than per-event — a reminder that idempotency design isn't one-size-fits-all; it depends on whether duplicate-suppression or fresh-resend is the actually desired behavior for that specific notification type.

---

## 8. Where it's applied — quick map

- Payment/order creation (Section 5)
- Notifications — email/SMS/push (Section 7)
- Webhook receivers (dedupe via event ID)
- Message queue consumers (Kafka, SQS, etc.)
- Database migrations / seed scripts
- Infra provisioning (AWS `ClientToken`, Terraform apply)
- Distributed locks / rate limiters (same atomic check-and-set class of problem — relevant to the Redis Lua-script rate limiter project)

---

## 9. Approaches — full stack view

### Client / Frontend

- Generate idempotency key (UUID v4) at the **click handler**, once per logical user intent — not regenerated inside retry loops.
- Disable the action button immediately on click — removes the trivial double-click-while-in-flight ambiguity at the UI layer, before it's even a backend concern.
- **Do not rely on in-memory/component state alone for surviving refresh** — a page reload destroys JS state entirely; the UUID is gone, frontend has no memory of the prior attempt.
- Two ways to survive refresh:
  - Persist a pointer (not a flag) in `localStorage`, scoped to the resource (`pending_purchase_<cartId>`), and re-check before allowing a new click.
  - **Better**: don't trust client storage at all — store nothing meaningful client-side beyond an ID/token; treat server as source of truth (see PaymentIntent model above). On every page load, query current status before enabling the action again.

### API / Gateway layer

- Idempotency key passed as a header (`Idempotency-Key`).
- Can be enforced at gateway level — reject POSTs without the header on sensitive routes (payments, order creation) before they even reach the backend.

### Backend / Application layer

- Atomic check-and-set on receiving a key — `SETNX` style claim (Redis), same race-condition class as a token-bucket rate limiter.
- Three states to explicitly handle:
  1. Key not seen → process, store result.
  2. Key seen, completed → return cached/stored response (no re-execution).
  3. Key seen, still processing (concurrent request racing in) → return 409 / block briefly, don't let two concurrent identical-key requests both execute.
- Store the result (status + body) with a TTL (Stripe: ~24h) so retries get a byte-identical response.

### Database layer

- Unique constraints as a backstop — `UNIQUE(idempotency_key)` or `UNIQUE(user_id, order_reference)` — protects even if application logic has a bug.
- Upserts (`INSERT ... ON CONFLICT DO UPDATE`) for idempotent writes without needing app-level locking at all.

### Messaging / async layer

- Idempotent producer config (Kafka: `enable.idempotence=true`).
- Idempotent consumer: dedup table of processed message IDs, or design the write itself to be naturally idempotent (`SET status=X` over `INCREMENT`).

---

## 10. One-line mental model to retain

> Retries are required for reliability. Idempotency is what makes retries safe. The HTTP spec gives you this for free on GET/PUT/DELETE — everything interesting happens because POST doesn't get it for free, and "exactly-once" is really just "at-least-once delivery + idempotent processing".
