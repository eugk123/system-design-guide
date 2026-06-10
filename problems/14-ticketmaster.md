# Problem 14: Ticket Booking (Ticketmaster / BookMyShow / seat reservation)

Tests **concurrency control on limited inventory** — the canonical "don't sell the same seat
twice" problem under a massive concurrent spike. The crux: **handling reservation races and
flash-sale load while keeping inventory correct.**

## Step 1 — Functional requirements
1. **Browse** events and available seats.
2. **Reserve / hold** a seat temporarily while the user pays.
3. **Book** (confirm) a seat — payment + permanent assignment.
4. Release holds that expire (user abandons checkout).
**Non-goals (defer)**: recommendations, dynamic pricing, the payment internals (see Problem 12).

## Step 2 — Non-functional requirements
- **Correctness / strong consistency on inventory**: a seat is sold to **exactly one** person —
  **no double-booking, no overselling.** This is the non-negotiable.
- **Handle massive concurrent spikes** — a hot concert: millions trying to buy thousands of seats
  in seconds. Read-heavy browse + intense write contention on a tiny inventory.
- **Availability** for browsing; **consistency** for booking (CP on the booking path).
- **Fairness** (queue/lottery) is often desired for hot events.

## Step 3 — Capacity estimation
Browsing: very high read QPS (everyone refreshing) → cache. Booking: the **contention** is the
issue, not raw throughput — thousands of seats, millions of concurrent attempts on the *same
small set of rows*. Classic **hot-row / hot-partition** problem.

## Step 4 — API
```
GET  /events/{id}/seats                  -> seat map + availability (cached, slightly stale OK)
POST /reservations  { eventId, seatIds } -> { reservationId, expiresAt }  (temporary hold)
POST /reservations/{id}/confirm  { paymentToken } -> { bookingId }        (after payment)
DELETE /reservations/{id}                 -> release hold
```

## Step 5 — High-level design
```
Client → LB → API → [ Booking service ]
                        │  inventory state machine: AVAILABLE → HELD → BOOKED
                        ▼
                Inventory DB (strongly consistent)  +  Redis (holds + locks + TTL)
                Payment service (Problem 12) ; Queue for confirmations/notifications
   Browse path: seat map served from cache (eventually consistent is fine for display)
```

## Step 6 — Deep dives

### Seat state machine
Each seat: **AVAILABLE → HELD (temporary, with TTL) → BOOKED (permanent)** — plus HELD → AVAILABLE
on expiry/release. All transitions must be **atomic and conflict-free**. This explicit lifecycle
is the backbone.

### Preventing double-booking — concurrency control (the heart)
Several valid approaches; know the tradeoffs:

**1. Pessimistic locking (DB row lock, `SELECT ... FOR UPDATE`)**
- Lock the seat row during the reserve transaction; others block/fail. Strong correctness.
- Cost: holding locks under a flash-sale spike serializes access → contention/latency; risk of
  long-held locks. Good for moderate contention.

**2. Optimistic concurrency (version / conditional update)**
- Update seat to HELD **only if** still AVAILABLE (`UPDATE ... WHERE status='AVAILABLE'`); if 0
  rows affected, someone beat you → retry/pick another seat. No long-held locks.
- Great when conflicts are relatively rare; under extreme contention many retries occur.

**3. Distributed lock / atomic hold in Redis (recommended for holds + TTL)**
- Atomically set a hold on the seat key with a **TTL** (e.g. `SET seat NX EX 600`). The **TTL is
  the elegant part**: if the user never pays, the hold **auto-expires** and the seat frees itself —
  no cron needed to clean abandoned carts. Use a **fencing token** so a stale holder can't confirm
  after expiry.
- The authoritative BOOKED state still lands in the strongly-consistent DB on confirm.

> Recommended: **Redis atomic hold with TTL for the temporary reservation** + a **conditional/
> transactional write to the inventory DB on final confirm** (the DB is the source of truth for
> BOOKED). This handles the hold lifecycle and the flash-sale spike while keeping booking correct.

### The hold → pay → confirm flow
1. Reserve: atomically move seats AVAILABLE → HELD with a short TTL; return `expiresAt`.
2. User pays within the window (Problem 12 — idempotent).
3. Confirm: verify the hold is still valid (fencing token), move HELD → BOOKED in the DB
   transactionally, finalize. If the hold expired → fail and ask the user to retry.
4. Abandon/expiry: hold TTL lapses → seat returns to AVAILABLE automatically.

### Handling the flash-sale spike (the load crux)
- **Virtual waiting room / queue**: admit users to the buying flow in **controlled batches** so
  the inventory layer isn't hit by millions simultaneously. Smooths the thundering herd and adds
  **fairness**. (This is what real ticketing sites do.)
- **Cache the seat map** for browsing (eventually consistent display); only the
  reserve/confirm path touches the consistent store.
- **Shard inventory by event**; a single hot event is still one hot partition → the waiting room +
  Redis holds absorb it. For huge venues, partition seats into sections.

### Consistency
- Booking path is **strongly consistent** (CP) — correctness over availability; better to show
  "try again" than to oversell.
- Display availability can be **eventually consistent** (a seat showing as available for a moment
  after being taken is fine — the atomic reserve will reject it).

## Step 7 — Bottlenecks & failure modes
- **Double-booking race** → atomic hold (Redis NX/TTL or DB conditional update) + DB as source of
  truth on confirm. *The headline.*
- **Abandoned carts** → TTL auto-release; no leaked inventory.
- **Flash-sale thundering herd** → virtual waiting room + queue + cached browse path.
- **Hot partition (one event)** → waiting room throttles; section-level partitioning.
- **Crash after payment, before confirm** → idempotent confirm + reconciliation; the payment's
  idempotency key lets you safely complete or refund.
- **Stale hold confirming late** → fencing token rejects expired holders.

## Key takeaways
- **Strong consistency on inventory — never oversell.** Booking is a CP, fail-safe path.
- **Atomic seat holds with a TTL** (Redis `SET NX EX` + fencing, or DB conditional update) handle
  both the reservation race and abandoned-cart cleanup elegantly.
- **Virtual waiting room / queue** is the standard answer to the **flash-sale spike** + fairness.
- Separate the **eventually-consistent browse path** (cached) from the **strongly-consistent
  booking path** (DB source of truth).
