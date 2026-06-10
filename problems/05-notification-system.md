# Problem 5: Notification System (Push / Email / SMS)

A fan-out + third-party-integration + reliability problem. The crux: **decoupling via queues,
delivering reliably through flaky external providers, and not spamming users.**

## Step 1 — Functional requirements
1. Send notifications across **channels**: mobile push (APNs/FCM), SMS, email, in-app.
2. Support **triggered** (event-driven) and **scheduled** notifications.
3. Respect **user preferences** (opt-in/out per channel/type) and **rate limits** (no spam).
4. **Templating** and per-user personalization.
**Non-goals (defer)**: full campaign/marketing UI, deep analytics dashboards.

## Step 2 — Non-functional requirements
- **High throughput** (millions–billions/day) with **fan-out**.
- **Reliability / at-least-once** delivery; never silently drop a critical notification.
- **Low latency for time-sensitive** notifications (OTP, security alerts); batch/throughput for
  the rest.
- **Availability** despite flaky third-party providers.
- **Idempotency** — never send the same notification twice.

## Step 3 — Capacity estimation
1B notifications/day → **~12K/s** avg, peak ~50K/s. Spiky (a broadcast to 100M users = a
massive burst) → the queue must absorb spikes. Storage: notification log + delivery status for
audit/retry, tiered retention.

## Step 4 — API
```
POST /notifications  { userId|segment, type, channel?, templateId, data, priority, sendAt? }
                     -> { notificationId }
```
Often the real "API" is **events** consumed off a bus (e.g. "order_shipped" → notification),
not just direct calls.

## Step 5 — High-level design
```
Producers (services/events) → Notification API/Ingestion
        → enqueue → [ Notification Service ]
                        │  (dedupe, fetch prefs, rate-limit, render template)
                        ▼
              Per-channel queues (push / sms / email)  ← decouple per channel
                        ▼
              Channel workers/senders → 3rd-party providers (APNs, FCM, Twilio, SES)
                        ▼
              Delivery status / retries / DLQ ; analytics events
```

### Flow
1. **Ingest** a request/event; validate; assign `notificationId`; **dedupe** (idempotency key).
2. **Resolve recipients & preferences**: expand a segment to users; drop users who opted out;
   apply **per-user rate limits / quiet hours** (don't notify at 3am, cap per day).
3. **Render** the template with user data; pick channel(s) by preference + priority.
4. **Enqueue** into the appropriate **per-channel queue** (Kafka/SQS). Returns fast.
5. **Channel workers** pull and call the **third-party provider**; record delivery status;
   retry transient failures; DLQ permanent failures.

## Step 6 — Deep dives

### Why per-channel queues + workers
- **Decoupling & spike absorption**: a 100M-user broadcast just fills the queue; workers drain
  at a sustainable rate; the rest of the system stays healthy (load leveling).
- **Independent scaling & isolation**: email being slow shouldn't delay OTP push. Separate
  queues (and even separate priority queues per channel) isolate failures and let each scale to
  its provider's throughput.
- **Provider rate limits**: APNs/FCM/Twilio impose quotas; workers throttle to stay within them
  (token bucket per provider) and back off on 429s.

### Priority
Two lanes: a **high-priority** path (OTP, security — low latency, jump the queue) and a
**bulk/low-priority** path (marketing, digests — throughput-optimized, can be delayed/batched).
Separate queues or priority queues.

### Reliability & idempotency (the heart of it)
- **At-least-once** delivery: workers retry on transient provider failures with exponential
  backoff + jitter; **DLQ** after N attempts for inspection/manual replay.
- **Idempotency / dedup**: each notification has a unique key; track sent IDs (dedup store with
  TTL) so retries and duplicate events don't double-send. Critical — users hate duplicate
  pushes, and double-sending an OTP is a security/UX problem.
- **Provider failover**: multiple providers per channel (e.g. two SMS vendors); on sustained
  failure, **circuit-break** and route to the backup.
- **Delivery tracking**: persist status (queued/sent/delivered/failed) per notification;
  consume provider webhooks/callbacks for delivered/bounced; feed retries and analytics.

### User preferences & anti-spam (what makes it production-grade)
- **Preference service**: per user, per type, per channel opt-in/out; honored before sending.
- **Rate limiting / frequency capping** per user (e.g. max N marketing/day), **quiet hours**,
  **deduplication of similar notifications**, and **digesting** (batch many events into one
  summary). This is often what interviewers probe — a senior engineer thinks about *not annoying
  users*, not just delivery mechanics.
- **Unsubscribe / compliance** (CAN-SPAM, GDPR/consent) for email/SMS.

### Templating
Template service with versioned templates + localization; render per-user with personalization
data. Cache compiled templates.

### Scheduled / delayed
A scheduler (delay queue, or a time-bucketed store polled by a dispatcher) enqueues at `sendAt`.
For "send to 100M at 9am," pre-expand and stage so the burst is smoothed.

## Step 7 — Bottlenecks & failure modes
- **Spike from broadcasts** → queues absorb; workers autoscale on queue depth; rate-limit egress
  to providers.
- **Provider outage** → circuit breaker + failover provider + retry with backoff; DLQ for the
  rest; never block the pipeline on one sick provider.
- **Duplicate events** (at-least-once upstream) → idempotency keys + dedup store.
- **Hot user / segment overlap** → frequency capping prevents one user getting hammered by
  multiple triggers.
- **Poison messages** → DLQ after N retries.
- **Observability** → track per-channel/provider success rate, latency, queue depth, bounce
  rate; alert on delivery-rate drops (a silent provider failure is the scary one).

## Key takeaways
- **Queue-centric, per-channel decoupling** is the backbone — absorbs spikes, isolates failures,
  scales independently.
- **Reliability = at-least-once + idempotency + retries/backoff + DLQ + provider failover.**
- **User preferences, rate limiting, quiet hours, and digesting** are first-class — the senior
  differentiator is caring about *not spamming users*, not just shipping bytes.
