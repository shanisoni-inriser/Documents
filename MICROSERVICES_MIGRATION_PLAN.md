# Victory Platform — Safe Microservice Migration Plan
## Splitting the Notification System and the Live Market Data System into separate servers — without breaking anything

**Date:** 2026-06-12
**Promise of this document:** every file path and line number written here was verified by reading the actual source code in `D:\Backend\new_backend\src`. Nothing is assumed. You can open any reference and check it yourself.

---

## Table of Contents

1. [What we are doing, in five sentences](#1-summary)
2. [The five safety locks — how we make sure nothing breaks](#2-safety)
3. [The system today — one picture](#3-today)
4. [Hidden connections in the code — the facts this whole plan is built on](#4-facts)
5. [The target architecture — where we are going](#5-target)
6. [The gateway (nginx) — the first and most important piece](#6-gateway)
7. [Notification Service — full design](#7-notif)
8. [Live Data Service — full design](#8-livedata)
9. [Databases, Redis, and the "who owns what" rules](#9-data)
10. [Authentication](#10-auth)
11. [The migration, phase by phase](#11-phases)
12. [Risk register](#12-risks)
13. [Monitoring and alerts](#13-observability)
14. [Testing strategy](#14-testing)
15. [Cutover and rollback runbooks](#15-runbooks)
16. [Folder structure and deployment](#16-structure)
17. [Things that will bite you — the checklist of classic mistakes](#17-bites)
18. [Final implementation order — one page](#18-order)

---

<a name="1-summary"></a>
## 1. What we are doing, in five sentences

1. **First we put a "traffic director" (nginx) in front of everything**, so that the website and the mobile app keep talking to one single address forever. After that, moving any feature to a new server is a one-line config change — and undoing it is flipping that line back.
2. **We extract the Notification Service first** (it is the lower-risk one). The monolith keeps a small, drop-in "NotificationClient" file, because three business flows inside the monolith fire notifications directly from code (trade auto-exit, trade expiry, plan changes) — and they must keep working exactly as today.
3. **We extract the Live Data Service second**, moving the *entire* tick pipeline — including the candle-writing part (`realtimeAggregator`), which needs **write** access to the database. Skipping that one file would silently kill all chart data.
4. **We never run two copies of the broker connections at the same time.** Instead, we switch over during a weekend evening when the market is closed — zero live ticks means zero chance of missing anything.
5. **We never delete old code on switch-over day.** The old path stays sleeping behind a switch for 2–4 weeks. Only after the new services have survived real production weeks do we delete anything.

---

<a name="2-safety"></a>
## 2. The five safety locks — how we make sure nothing breaks

First, an honest sentence: no engineer on earth can promise a literal "100%" — anyone who does is selling something. What we **can** do — and what this plan does — is design the migration so that:

- every risky step is **checked against recorded reality** before real users ever touch it, **and**
- every step can be **undone in seconds to minutes**, **and**
- any problem that somehow slips through is **caught by an alarm, not by a customer**.

When all three are true at every step, the practical risk of losing functionality is as close to zero as production engineering allows. This plan achieves it with **five independent locks** — even if one lock somehow fails, the other four still protect you. This is the same pattern that large SaaS companies use for exactly this kind of work.

**🔒 Lock 1 — Clients never change, ever.**
The web admin panel and the mobile app keep talking to one single address forever. All the "moving" happens behind the curtain (see §6). A client that doesn't change cannot break.

**🔒 Lock 2 — Every step has a seconds-level undo.**

```
  THE ROLLBACK LADDER — how fast can we undo, at any moment?

  Phase 0   nginx installed (pass-through)  →  point DNS back............(minutes)
  Phase 1   message pipe inside monolith    →  flip 1 switch.............(seconds)
  Phase 2   notif routes moved to :3005     →  flip 1 nginx line.........(seconds)
  Phase 2   consumer + cron flipped         →  flip 2 switches...........(< 1 min)
  Phase 3   live-data switch-over           →  4 steps in reverse........(< 5 min)
  Bake      is old code deleted? NO         →  old path redeployable.....(minutes)
```

There is **no point in this entire migration** where going back takes longer than a coffee break.

**🔒 Lock 3 — We compare against recorded reality, not against hope.**

> **📘 New concept — "golden files":** before changing anything, we record exactly what the system sends today — the WebSocket messages, the API responses, the Redis values, the database rows. These recordings are called *golden files*. Later, the new service must produce **byte-for-byte the same output** from the same input. "Nothing changed" stops being an opinion and becomes a measured fact.

```
  Phase 0: RECORD what the system               Later: COMPARE the new service
  does today (the golden files)                 against those recordings

  ┌──────────────────────┐                      ┌────────────────────────────┐
  │  live system today   │                      │  new service (in testing)  │
  │   WS messages ───────┼──► saved files  ───► │   same inputs replayed     │
  │   REST responses ────┼──► saved files       │   outputs compared to files│
  │   Redis values ──────┼──► saved files       │                            │
  │   candle DB rows ────┼──► saved files       │   ANY difference =         │
  └──────────────────────┘                      │   STOP → fix → re-test     │
                                                └────────────────────────────┘
```

**🔒 Lock 4 — Only one thing changes at a time.**
Every phase moves exactly one part while everything else stays frozen. If anything misbehaves, there is exactly **one suspect** — no guessing, no debugging two changes at once at 9:15 AM.

**🔒 Lock 5 — Old code is never deleted until the new path has survived weeks of real production.**
During the bake period (see box below) the monolith can take back any job within minutes.

> **📘 New concept — "bake period":** after a switch-over, we keep the old code in place (turned off, but ready) for 2–4 weeks while the new service handles real traffic. Like letting a cake bake fully before you take it out — we don't declare success on day one.

### 2.1 What this plan will NEVER do (hard rules)

1. **No AI tool runs any database query — ever.** Nothing in this migration was executed against your database, and nothing will be. The only SQL in this entire plan is **three additive statements** (§9.5) that **your team runs by hand**, after taking a backup.
2. **No existing data is modified or deleted by the migration.** No `DROP`, no `DELETE`, no `UPDATE` of user data — the three statements only *add* (two columns, one constraint, one read-only login).
3. **No URL, payload shape, event name, or WebSocket message ever changes.** The contracts in §8.2 are frozen and verified by golden files.
4. **No two broker connections at the same time** — running the same broker login from two processes makes them log each other out (§8.4).
5. **No business-logic refactoring during the move.** Code moves as-is. Only the explicitly listed import swaps (§7.2) and two small adapters (§8.5, §8.6) change — each named, file by file, line by line.

---

<a name="3-today"></a>
## 3. The system today — one picture

```
THE SYSTEM TODAY — everything inside the big box is ONE process on :3002

  Admin Panel (web)            Mobile App (user phones)       Strategy Engine :3003
      │ REST + /ws/prices          │ FCM pushes + REST            (already separate —
      ▼                            ▼                               we do NOT touch it)
┌──────────────────────────────────────────────────────────────────┐
│                        MONOLITH  :3002                           │
│                                                                  │
│   ~140 admin/API routes      GraphQL       auth / sessions       │
│                                                                  │
│   NOTIFICATIONS                     LIVE MARKET DATA             │
│   • admin "send now"                • broker connections         │
│   • scheduled (every-minute cron)     (Angel One, Dhan)          │
│   • trade auto-exit pushes          • TickGate validation        │
│   • plan-change pushes              • WebSocket broadcast        │
│   • Firebase (FCM) sending         • 1-min candle WRITING        │
│         │                                 │              │       │
└─────────┼─────────────────────────────────┼──────────────┼───────┘
          ▼                                 ▼              ▼
   victory_db (Postgres)              Redis cache    victory_market_db
   users, notifications…           PRICE:LATEST:*    (TimescaleDB candles)

  ⚠ Important fact: today, a crash ANYWHERE in those ~140 routes kills the
    live market feed for every user, because index.js (line 464) does
    process.exit(1) on any uncaught exception. Separating the services
    FIXES this — it is one of the strongest reasons to do this migration.
```

> **📘 New concept — FCM (Firebase Cloud Messaging):** Google's free service for sending push notifications to phones. Your backend gives Firebase a message + a device token, and Firebase delivers it to the user's phone. Each user's phone registers a *token* (stored in your `push_devices` table).

> **📘 New concept — tick:** one live price update from the stock broker — "RELIANCE is now ₹2,850.50". During market hours, thousands of ticks per minute flow through the system.

### 3.1 The FIVE places that fire notifications (knowing all five is critical)

```
PATH 1  Admin clicks "Send Now"  → POST /api/notification/send → broadcast service → FCM
PATH 2  Scheduled notification    → every-minute cron checks date+time → send → FCM
PATH 3  Trade hits target/SL      → tradeRecommendations.controller (lines 1218/1397/1451)
                                    → broadcast service → FCM
PATH 4  Trade expires             → tradeExpireCron → autoCloseExpiredTrades
                                    → broadcast service (line 1502) → FCM
PATH 5  User's plan changes       → plansNew.controller:74 / proPlanTransactions:90,155
                                    → planSync.service → Firebase directly (silent push)
```

Paths 1 and 2 are the "obvious" ones. **Paths 3, 4 and 5 start inside the monolith's own business logic** — they are the reason the monolith must keep a small notification client after the split (§7.2). Forget any one of them, and that flow silently stops sending pushes.

### 3.2 The live data flow today (with the part people usually miss, in bold)

```
BROKER (Angel One / Dhan)
   ↓ WebSocket connection
Broker adapter (parses the raw feed into a clean tick)
   ↓
   ├─→ TickGate (8 sanity checks) → Redis SET PRICE:LATEST:{symbol} (60s expiry)
   │                              → push to /ws/market subscribers
   ├─→ broadcast to ALL /ws/prices clients (what the admin panel uses)
   └─→ **realtimeAggregator: collects ticks into 1-minute candles
        and WRITES them into TimescaleDB on a flush timer**   ← this is what
                                                                 charts/history
                                                                 are built from
```

> **📘 New concept — candle (OHLC):** charts don't store every tick. They store, per minute: the Open, High, Low and Close price (plus volume) — one "candle" per minute. The `realtimeAggregator` is the worker that builds these candles from live ticks and saves them to the database.

### 3.3 The clients — and which ones we can never break

| Client | Uses | Can we update it instantly? |
|---|---|---|
| Admin panel (React web app) | REST `/api/*`, WebSocket `/ws/prices` | Yes — it's a website |
| **Mobile app (on user phones)** | FCM pushes, user-feed endpoints (`getNotificationUserWise`, `mark-delivered`, `mark-opened`, `save-token`…) | **NO — it is installed on phones. Its URLs are frozen.** |
| Strategy Engine (:3003) | Redis events, shared JWT secret | Yes, but it needs zero changes |

The mobile app is the single most important constraint in this whole plan: **we can never move a URL it uses.** That is why the gateway (§6) comes first.

---

<a name="4-facts"></a>
## 4. Hidden connections in the code — the facts this whole plan is built on

A migration breaks things when it misses a hidden connection. Below are the six hidden connections found by reading the actual source code. Each one shapes a specific part of this plan. **If you remember nothing else from this document, remember these six.**

### Fact 1 — Trade notifications are fired from inside the trade controller

`src/controllers/tradeRecommendations.controller.js` lines 6–7 import the broadcast service directly, and call it at lines **939, 956, 973, 989** (inside `sendAutoExitNotifications()`) — this is what notifies users when a trade hits **target, stop-loss, or expiry**. It is triggered from four places (lines 1218, 1397, 1451, 1502), including the expiry cron.

**Danger:** these calls are wrapped in `try/catch` that only does `console.error`. If notification code is simply moved out of the monolith, these pushes don't crash — **they silently stop arriving**, and you find out from angry users.
**Plan response:** the `NotificationClient` (§7.2).

### Fact 2 — Plan-change pushes use Firebase directly, from billing code

`src/services/planSync.service.js` line 2 imports Firebase directly and sends silent pushes that tell the mobile app a user's plan changed. It is called from `plansNew.controller.js:74` and `proPlanTransactions.controller.js:90, 155` — billing flows that stay in the monolith.

**Danger:** remove Firebase credentials from the monolith and **plan purchases stop syncing to phones** — silently (errors only go to `console.error` at lines 65, 113).
**Plan response:** same `NotificationClient` (§7.2), one more import swap.

### Fact 3 — The tick pipeline WRITES to the database (it is not read-only)

`src/services/marketData/realtimeAggregator.service.js` is imported by the Angel One tick path itself (`angelWebSocket.service.js:6`, `angelWebSocket.service.v2.js:7`). It builds 1-minute candles and **writes them to TimescaleDB**: parallel inserts (line 54), `insert1MinCandle` (line 218), on a flush timer (line 25).

**Danger:** if the live data system is extracted with a "read-only" database connection — which sounds safe but is wrong — every candle insert fails, and charts/history/movers **slowly go stale during market hours** while live prices still flash on screen (so a quick test looks fine).
**Plan response:** the aggregator moves *with* the tick path, with a **read/write** pool (§8.1).

### Fact 4 — A monolith file imports from the folder being moved out

`src/services/angel.service.js` line 2 — a file that **stays** in the monolith (it serves historical candle data) — does `require('./marketData/angelAuth.service.js')`, a file that **moves** to the live data service.

**Danger:** after the move, the monolith's `require()` points at a missing file and **the whole monolith refuses to start**.
**Plan response:** a small "token reader" adapter in the monolith (§8.5), shipped *before* switch-over day.

### Fact 5 — The broker status endpoint reads in-process memory

`src/controllers/broker.controller.js` lines 1, 5 call `getBrokerManager()` — a singleton that lives in the same process as the brokers. After the split, this returns a fresh **empty** manager.

**Danger:** the admin panel's broker status page would confidently report "no brokers" — not an error, a **lie**.
**Plan response:** that controller becomes a tiny proxy to the live data service's `/health` (§8.6).

### Fact 6 — The order-depth feed appears to be dead code

`src/services/smartapiStream.js` (the order-depth feed) is **imported by no file in `src/`** — the only mentions are comments inside `Migration/028_fix_on_conflict_constraints.sql`. It is never started from `index.js`. Meanwhile `indexCalculation.service.js` reads `Order_Depth:*` Redis keys that this file was supposed to write.

**Plan response:** Phase 0 settles whether order-depth works at all today, *before* anyone spends migration effort on it — and before anyone blames the migration for a feature that was already dead (§11, Phase 0.5).

---

<a name="5-target"></a>
## 5. The target architecture — where we are going

```
                        ┌──────────────────────────────────────────────┐
   Admin Panel (web)    │           NGINX  (one public address)         │   Mobile App
   ─────────────────────►  routes by URL path:                          ◄──────────────
                        │   /ws/prices, /ws/market      → live-data :3004
                        │   /api/notification*           → notif-svc :3005
                        │   /api/notifications*          → notif-svc :3005
                        │   /api/notification-query*     → notif-svc :3005
                        │   everything else              → monolith  :3002
                        └───────┬──────────────┬──────────────┬─────────┘
                                │              │              │
                 ┌──────────────▼───┐  ┌───────▼────────┐  ┌──▼──────────────────┐
                 │ MONOLITH :3002   │  │ NOTIF SVC :3005│  │ LIVE-DATA SVC :3004 │
                 │ auth, users,     │  │ all notif      │  │ brokers, TickGate,  │
                 │ billing, ~140    │  │ routes + crons │  │ WebSocket serving,  │
                 │ admin routes,    │  │ FCM (Firebase) │  │ candle WRITING,     │
                 │ GraphQL, trades, │  │ message-pipe   │  │ market scheduler,   │
                 │ content          │  │ consumer       │  │ Angel login OWNER   │
                 │                  │  │                │  │                     │
                 │ NotificationClient──┼──► Redis Stream │  │                     │
                 │ (tiny, drop-in)  │  │  notif:dispatch│  │                     │
                 │ Angel token      │  └───────┬────────┘  └──┬──────────────────┘
                 │ READER           │          │              │
                 └────────┬─────────┘          │              │
                          │                    │              │
        ┌─────────────────▼────────────────────▼──────────────▼───────────┐
        │  PostgreSQL: victory_db + victory_market_db (shared, with clear │
        │  "who owns which table" rules — see §9)                         │
        │  Redis: PRICE:LATEST:* · notif:dispatch · ANGEL:SESSION         │
        └──────────────────────────────────────────────────────────────────┘

   Strategy Engine :3003 — UNTOUCHED (it is already a separate service)
```

How the services talk to each other:

| From → To | Style | Mechanism | Why |
|---|---|---|---|
| Clients → all services | direct request | HTTP/WebSocket **through nginx only** | URLs frozen forever, instant re-routing |
| Monolith → Notif svc (trade/plan/expiry triggers) | **fire-and-forget message** | Redis Stream `notif:dispatch` | if the notif service is down, messages **wait safely** and are processed when it returns — the trade flow never gets stuck waiting for Firebase |
| Admin panel → Notif svc (create/send/list) | direct request | HTTP via nginx | the admin needs an immediate answer |
| Live-data → everyone | fire-and-forget | Redis keys + WebSocket broadcast | already the contract today |
| Monolith → Live-data (broker status) | direct request | internal HTTP `/health` | tiny, read-only |
| Anything → Angel One login | **only the live-data service** | — | single-owner rule (§8.5) |

> **📘 New concept — "fire-and-forget":** the sender drops a message in a mailbox and continues its work immediately. It does not stand at the mailbox waiting for a reply. This is exactly right for notifications: a trade closing should never be delayed because Firebase is slow.

---

<a name="6-gateway"></a>
## 6. The gateway (nginx) — the first and most important piece

> **📘 New concept — nginx (a "reverse proxy" / gateway):** a small, extremely fast and battle-tested program that sits at your public address and **forwards each request to the right server behind it**, based on the URL. Think of it as a hotel receptionist: every guest walks up to the same front desk, and the receptionist quietly directs them to the right room. The guests (your web app, your mobile app) never need to know the room numbers — and you can move things between rooms without telling any guest.
>
> ```
>                              ┌──► room :3002  (monolith)
>   all clients ──► reception ─┼──► room :3004  (live data)
>                   (nginx)    └──► room :3005  (notifications)
>
>   Clients only ever know the reception desk's address.
>   Moving a feature to a new room = telling the receptionist, not the guests.
> ```
>
> nginx is free, runs everywhere (including Windows), handles WebSockets natively, and adds well under a millisecond of delay — your ticks already spend tens of milliseconds traveling the internet, so this cost is invisible.

We deploy nginx **before any extraction**, routing 100% of traffic to the monolith. Pure pass-through, zero behavior change — and it is the single highest-leverage step of the migration:

- **Zero client changes, forever.** Web and mobile keep one address. (Remember §3.3: mobile URLs are frozen — this is the only honest way to respect that.)
- **Per-route canary.** Move one small endpoint group to the new service while everything else stays put.
- **Instant rollback.** Switch-over and rollback are a one-line change + reload (sub-second, no deploys, no client involvement).

> **📘 New concept — canary release:** named after the canary birds miners took underground — a small early-warning test. Instead of moving *all* notification routes at once, we move the smallest, least risky group first and watch it for a day. If anything is wrong, only that tiny group was exposed, and we flip it back in seconds.

Reference config (the two WebSocket settings are the ones people forget):

```nginx
upstream monolith      { server 127.0.0.1:3002; }
upstream notif_svc     { server 127.0.0.1:3005; }
upstream livedata_svc  { server 127.0.0.1:3004; }

map $http_upgrade $connection_upgrade { default upgrade; '' close; }

server {
    listen 80;  # add HTTPS/TLS in production

    # WebSockets → live-data service (after Phase 3; before that → monolith)
    location ~ ^/ws/(prices|market)$ {
        proxy_pass http://livedata_svc;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # MUST be longer than the server's 30s ping interval,
        # or nginx will cut healthy idle connections:
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }

    # Notifications → notification service (after Phase 2)
    location ~ ^/api/(notification|notifications|notification-query) {
        proxy_pass http://notif_svc;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Correlation-Id $request_id;
    }

    # Everything else → monolith
    location / {
        proxy_pass http://monolith;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 50m;   # the monolith accepts 50 MB uploads today
    }
}
```

Notes:
- Login cookies pass through untouched (same address → same cookie rules). Authentication keeps working everywhere with zero changes.
- If you later run a second live-data instance, plain WebSockets need no "sticky sessions" — clients simply re-subscribe on reconnect, which the frontend already does.

---

<a name="7-notif"></a>
## 7. Notification Service — full design

### 7.1 What moves to the new service

`push.service.js`, `notificationBroadcast.service.js`, `autoNotification.service.js`, `autoNotificationCron.js`, the three notification controllers, the three notification route files, `config/firebase.js` + the Firebase service-account JSON — plus **copies** of `authenticateUser.js`, `authorizeAccess.js`, cookie-parser, and the exact same CORS settings.

### 7.2 What the monolith KEEPS: the `NotificationClient` (this protects Facts 1 & 2)

One new small file in the monolith. It exports **the same function names with the same signatures** as the code that is moving out — so the business logic doesn't know anything changed:

```js
// src/services/notificationClient.js  (stays in the monolith)
const { getRedis } = require("../db/redisClient");
const { randomUUID } = require("crypto");

async function dispatch(action, payload, correlationId) {
  await getRedis().xAdd("notif:dispatch", "*", {
    action,                                  // "BROADCAST_BY_QUERY" | "PLAN_SYNC" | "USER_PLAN_SYNC"
    payload: JSON.stringify(payload),
    idempotency_key: payload.idempotencyKey || randomUUID(),
    correlation_id: correlationId || randomUUID(),
    ts: Date.now().toString(),
  });
}

// Drop-in replacements — same names and signatures as the old in-process functions:
exports.broadcastNotificationByQuery = (args) => dispatch("BROADCAST_BY_QUERY", args);
exports.notifyPlanChange            = (args) => dispatch("PLAN_SYNC", args);
exports.notifyUserPlanChange        = (args) => dispatch("USER_PLAN_SYNC", args);
```

> **📘 New concept — Redis Stream (the "message pipe"):** Redis is the in-memory data store you already run. A *Stream* is its built-in durable message queue: one side appends messages (`XADD`), the other side reads them in order. The key property: **if the reader is down, messages don't disappear — they wait in the pipe** until the reader comes back. That is exactly what notifications need.
>
> ```
>   monolith ──XADD──►  ║▓▓▓▓▓░░░░║  ──XREADGROUP──► notification service
>                        the pipe                       (reads, sends push,
>                   (messages wait here                  then ACKnowledges)
>                    if reader is down)
> ```

With this client in place, exactly **four import lines change** in the whole monolith — and *zero business logic*:

| File | Old import | New import |
|---|---|---|
| `controllers/tradeRecommendations.controller.js:6-7` | `require("../services/notificationBroadcast.service.js")` | `require("../services/notificationClient.js")` |
| `controllers/plansNew.controller.js:74` | `require("../services/planSync.service")` | `require("../services/notificationClient.js")` |
| `controllers/proPlanTransactions.controller.js:90` | same | same |
| `controllers/proPlanTransactions.controller.js:155` | same | same |

`tradeExpireCron.js` needs no change — it already goes through the controller.

One honest nuance: these triggers become *asynchronous* — the message is queued, then sent by the service typically within milliseconds. For phone pushes this is invisible (Firebase delivery itself takes seconds). None of the call sites use the send result for business decisions — they only log errors. (Verify this during Phase 1 by checking each call site.)

### 7.3 The service side: the consumer

> **📘 New concept — consumer group + ACK:** a *consumer group* is Redis's way of letting one or more workers share a stream safely. A worker reads a message, processes it, then sends an **ACK** ("done"). If a worker crashes before ACKing, Redis keeps the message in a "pending" list and another worker can pick it up. Nothing is lost.

```js
// notification-service/src/consumers/dispatchConsumer.js
// One-time setup: XGROUP CREATE notif:dispatch notif_workers $ MKSTREAM
while (running) {
  const msgs = await redis.xReadGroup("notif_workers", consumerName,
      { key: "notif:dispatch", id: ">" }, { COUNT: 10, BLOCK: 5000 });
  for (const m of msgs) {
    try {
      await handle(m);                      // routes on m.action → the existing service functions
      await redis.xAck("notif:dispatch", "notif_workers", m.id);
    } catch (err) {
      // after 5 failed attempts, park the message in the dead-letter stream:
      await redis.xAdd("notif:dispatch:dead", "*", { ...m, error: String(err) });
      await redis.xAck("notif:dispatch", "notif_workers", m.id);
      log.error({ correlationId: m.correlation_id }, "moved to DLQ");
    }
  }
}
```

> **📘 New concept — DLQ (dead-letter queue):** a parking lot for messages that keep failing. Instead of retrying a broken message forever (and blocking everything behind it), we move it aside after 5 attempts, raise an alarm, and a human looks at it. No message is ever silently thrown away.

- Delivery guarantee is **at-least-once** (a crash before ACK means the message is delivered again). That is why the next section exists.
- A startup sweeper (`XAUTOCLAIM`, idle > 60s) rescues messages stuck with a dead worker.

### 7.4 Real duplicate-protection (idempotency)

> **📘 New concept — idempotency:** an operation is *idempotent* if doing it twice has the same effect as doing it once. Since our message pipe can deliver a message twice (at-least-once), the *receiver* must make duplicates harmless. We do it at the database level — the only level that cannot be bypassed by a bug, a crash, or a double-running cron.

Two database-level guarantees:

**1. Atomic claim before sending a scheduled notification:**

```sql
UPDATE notifications
SET    status = 'PROCESSING', claimed_by = $1, claimed_at = now()
WHERE  id = $2 AND status = 'PENDING'
RETURNING id;
-- 0 rows returned ⇒ someone else already claimed it ⇒ skip.
-- Even two crons running at once cannot both claim the same row.
```

A sweeper resets rows stuck in `PROCESSING` for more than 10 minutes back to `PENDING` (crash recovery); the constraint below stops re-sends to users who already received it.

**2. A unique constraint as the duplicate firewall:**

```sql
ALTER TABLE push_notifications
  ADD CONSTRAINT uq_push_notif_user UNIQUE (notification_id, user_id);
-- The sender does INSERT ... ON CONFLICT DO NOTHING, and only pushes to FCM
-- for rows that were actually inserted.
```

With these two, a double-running cron or a re-delivered message produces **zero** duplicate pushes — by construction, not by hope.

*(Both statements are run by your team, by hand, after a backup — see the database safety rules in §9.5.)*

### 7.5 Cron ownership

> **📘 New concept — cron:** a scheduler that runs a task on a fixed timetable (e.g., "every minute", "8:45 every weekday"). Your system already uses `node-cron` for scheduled notifications, backups, and token refresh.

The every-minute scheduled-notification cron moves to the notification service. During the transition both processes *may* briefly run it — the database claim above makes that completely harmless. One operational must: the new service's machine must be on **Asia/Kolkata** time (or the code's explicit IST conversion must be verified) — a UTC clock would shift every scheduled notification by 5.5 hours.

### 7.6 The service's public API (unchanged paths — nginx makes that possible)

All existing paths stay byte-identical: `/api/notification/send`, `/api/notification/getNotificationUserWise`, `mark-delivered`, `mark-opened`, delete, the admin CRUD under `/api/notifications/`, and the saved-targeting-queries CRUD under `/api/notification-query/`. The frontend and the mobile app notice **nothing**.

---

<a name="8-livedata"></a>
## 8. Live Data Service — full design

### 8.1 The complete move list

**Moves to the live-data service:**

| Component | Note |
|---|---|
| `sockets/` — all 8 files | wsHub, priceSocket, index, internalMarket.websocket + instance, angelOne instance, sharekhan ×2 |
| `core/` — all 6 files | BaseBrokerWS, BrokerManager, BrokerRegistry, TickGate, MessageNormalizer, MarketScheduler |
| `brokers/dhan/` — all 4 files | self-contained |
| `services/marketData/angelAuth.service.js`, `angelLoginRunner.js`, `angelTokenScheduler.js`, `angelWebSocket.service.js`, `angelWebSocket.service.v2.js` | the live-data service becomes the **only** process that logs in to Angel One |
| **`services/marketData/realtimeAggregator.service.js`** | the candle writer — moves with the tick path and gets a **read/write** DB pool (Fact 3) |
| `services/marketData/tickGenerator.service.js` | verify its consumers first; if it feeds the same pipeline, it moves |
| `config/brokerConfig.js`, `script/syncAngelOneTokens.js`, and the 8:45 AM token cron from `index.js` | the token lifecycle moves as one unit |
| copies of `db/redisClient.js`, `db/timescaleClient.js` | own connections |

**Stays in the monolith (these only READ market data):**

| Component | Why it stays |
|---|---|
| `intervals.controller.js`, `history.controller.js`, `movers.controller.js` + their services | read candles from the database |
| `price.controller.js` + `price.service.js` | read `PRICE:LATEST:*` from Redis |
| `indices.controller.js` + `indexCalculation.service.js` | read Redis |
| `angel.service.js` | historical candles — needs the token-reader adapter (§8.5) |
| `broker.controller.js` | becomes a proxy (§8.6) |
| chart/sparkline/symbols/trading routes | database reads |

**Decide before moving:** `services/smartapiStream.js` — see Fact 6. If order-depth is dead, this file is deleted, not migrated.

The dividing line, in one sentence: **everything fed by live ticks moves; everything that only reads stored results stays.**

### 8.2 The frozen contracts (this is what "nothing breaks" *means*)

Write these down, record real samples of each in Phase 0 (golden files), and compare at every later step:

1. **`/ws/prices` messages:** `{ "type": "price", "data": {...} }` and `{ "type": "depth", "data": {...} }` — broadcast to every connected client.
2. **`/ws/market` protocol:** client sends `{ type: "SUBSCRIBE", page, context, symbols[] }` / `{ type: "UNSUBSCRIBE_DELAYED", ... }` (30-second grace period); server replies with Redis snapshots immediately, then live `{ type: "PRICE", symbol, ... }` messages; only symbols enabled in `InternalTokenList.websocket_enabled` are broadcast.
3. **Redis keys:** `PRICE:LATEST:{symbol}` — exact same JSON shape, same 60s expiry. (`Order_Depth:{token}` only if Fact 6 says it's alive.)
4. **Database writes:** 1-minute candle rows exactly as `insert1MinCandle` produces them today — same table, same conflict handling.
5. **Auth:** both WebSocket endpoints are open (no login) today. Keep that — market prices are public data; do not "improve" security mid-migration.

### 8.3 A built-in path to future scaling (cheap now, priceless later)

Broker connections must always be a **single instance** — a broker session is a singleton; you cannot run it twice (see §8.4). So the only honest way to scale is to split *receiving* ticks from *distributing* ticks:

```
                          single instance                many instances (future)
   BROKERS ──► [ INGEST: broker connections,    ──►  [ FAN-OUT: WebSocket
                 TickGate, candle writing ]            servers for clients ]
                        │                                      ▲
                        └── Redis publish "ticks:validated" ───┘
```

We prepare this seam now at almost zero cost: inside the new service, `BrokerManager._onTick` *additionally* publishes each accepted tick to a Redis channel `ticks:validated` (behind a switch, off by default — one line, one Redis publish). Today, one process does both jobs, exactly like now. The day one fan-out process is not enough, you start more — without ever touching the broker code again.

### 8.4 The Angel One login: the single-owner rule (this protects Fact 4 and prevents session wars)

**Why this matters:** if two processes log in to Angel One with the same system credentials, each new login invalidates the other's session — they knock each other offline in a loop. This is also why this plan never "dual-runs" broker connections for testing.

```
   THE SINGLE-OWNER RULE

   live-data service  ──── logs in / refreshes (8:45 cron) ───► Angel One
        │
        └── writes the fresh token to:  credentials table  +  Redis ANGEL:SESSION

   monolith (angel.service.js) ──── READS the token only ──── never logs in
```

- **Owner:** the live-data service runs `AngelAuthService.login()`, the token scheduler, and the 8:45 AM IST refresh cron. Nobody else ever logs in with system credentials.
- **Monolith change:** `angel.service.js:2` swaps its import for a ~20-line `angelTokenReader.js` that reads the token from Redis/DB. If a monolith call gets a 401 from Angel One, it re-reads the token (the owner will have refreshed it) — it must **never** trigger a login itself.
- Per-user broker tokens (the `save-token` / place-order flows) are a different thing — they are user-supplied sessions and stay in the monolith untouched (verified: `angelOne.controller.js` does not use `AngelAuthService`).

### 8.5 The token-reader adapter (the Fact-4 fix, ~20 lines)

Shipped to the monolith *before* switch-over day, switched on by the `LIVE_DATA_EXTRACTED` flag.

### 8.6 The broker-status proxy (the Fact-5 fix)

```js
// monolith: controllers/broker.controller.js (after Phase 3)
exports.getBrokerStatus = async (req, res) => {
  const r = await axios.get(`${env.LIVE_DATA_URL}/health`, { timeout: 2000 });
  res.json(r.data.brokers);
};
```

The live-data service exposes the same shape `getBrokerManager().getHealth()` returns today, so the admin panel sees an identical response.

### 8.7 The slow-client guard (one line that prevents a classic crash)

> **📘 New concept — backpressure:** if one client's network is slow, the server's send buffer for that client grows and grows. With thousands of ticks, an unchecked buffer can eat all the server's memory and crash it — taking everyone down because of one bad connection.

```js
// in every broadcast loop:
if (ws.bufferedAmount > 1_000_000) return;  // skip this slow client;
                                            // the next tick supersedes this one anyway
```

Price ticks are "last value wins" by nature — skipping one tick to a stalled client is strictly better than buffering until the process dies.

---

<a name="9-data"></a>
## 9. Databases, Redis, and the "who owns what" rules

### 9.1 Ownership table (one writer per thing — always)

> **📘 New concept — single-writer rule:** for every table and every Redis key, exactly **one** service is allowed to write it. Everyone else may only read. This removes a whole category of bugs (two services overwriting each other) without splitting any database.

| Store / resource | The one writer | Readers |
|---|---|---|
| `OHLC_*` 1-min candles (victory_market_db) | **Live-data svc** (realtimeAggregator) | Monolith (charts/history/movers), Strategy Engine |
| `PRICE:LATEST:{symbol}` (Redis) | Live-data svc | Monolith REST, WebSocket layer |
| Angel system session (credentials table + `ANGEL:SESSION`) | **Live-data svc only** | Monolith `angel.service` (read-only) |
| `notifications`, `push_notifications`, `push_devices`, `notification_types`, `notification_query` | Notification svc | (monolith only writes via the message pipe) |
| `notif:dispatch` / `notif:dispatch:dead` (Redis Streams) | Monolith produces / Notif svc consumes | — |
| `InternalTokenList`, `control_tradingdays` | Monolith (admin UI writes) | Live-data svc polls them (30s / 15s) |
| `users`, permissions, billing, content tables | Monolith | Notif svc reads `users` for targeting (read-only role, §9.4) |

### 9.2 Connection-pool budget

> **📘 New concept — connection pool:** each service keeps a small set of reusable database connections (a "pool") instead of opening a new one per query. PostgreSQL allows ~100 connections total by default — so when several services share one database, their pool sizes must be budgeted like a household budget.

| Service | victory_db | victory_market_db | Reasoning |
|---|---|---|---|
| Monolith | 20 | 15 | unchanged primary writer |
| Notification svc | 5 | — | low write volume |
| Live-data svc | — | **10 read/write** | parallel candle inserts + polling reads |
| Strategy Engine | — | 20 | unchanged |
| **Total** | 25 | 45 | comfortably under the ~100 limit |

### 9.3 Why the databases stay shared (a deliberate choice)

Splitting databases during a service extraction would force data migration, double-writing periods, and replacing cross-table JOINs — three big risk multipliers — for zero functional gain. A shared database **with enforced single-writer rules** (§9.1) gives the real benefit (clear ownership) without the risk. Revisit a physical split only after both services have been stable for a quarter.

### 9.4 A security fix that comes free with the extraction

The `notification_query` table stores **raw SQL that the server executes** to choose notification recipients. Today those queries run with full database privileges — meaning a bad or malicious saved query could, in theory, do anything. While extracting, we give the notification service a second, locked-down database login used *only* for these targeting queries:

```sql
CREATE ROLE notif_targeting LOGIN PASSWORD '...';
GRANT SELECT ON users, push_devices /* + whatever targeting needs */ TO notif_targeting;
ALTER ROLE notif_targeting SET statement_timeout = '5s';
ALTER ROLE notif_targeting SET default_transaction_read_only = on;
```

Now the worst a bad targeting query can do is read some user rows slowly for 5 seconds. It can no longer modify or delete anything. This costs about an hour and removes the single scariest thing in the codebase.

### 9.5 Database safety rules (read this before touching anything)

**Rule zero: no query in this plan is ever run automatically.** No AI tool, no script, no migration step executes SQL against your database on its own. Every statement below is run **by a human on your team**, in the listed phase, **after taking a backup** (`pg_dump` of `victory_db`), in a quiet window.

The entire migration needs only **three** SQL statements — all *additive* (they only ADD things; they are physically incapable of losing data):

| # | Statement (defined in) | What it does | Can it lose data? |
|---|---|---|---|
| 1 | `ALTER TABLE notifications ADD COLUMN claimed_by …, claimed_at …` (§7.4) | adds two empty bookkeeping columns | No — adding columns never touches existing rows |
| 2 | `ALTER TABLE push_notifications ADD CONSTRAINT uq_push_notif_user UNIQUE (notification_id, user_id)` (§7.4) | adds a duplicate-blocker for the future | No — it only *refuses future duplicates*. If old duplicates already exist, it simply **fails to apply, harmlessly**, and changes nothing |
| 3 | `CREATE ROLE notif_targeting … GRANT SELECT …` (§9.4) | creates a new **read-only** database login | No — it creates something new and touches nothing existing |

**About old duplicates (statement 2):** if `push_notifications` already contains duplicate rows, **do not delete them directly**. The safe order, done by hand:

1. Take a backup.
2. Copy the duplicates aside first: `CREATE TABLE push_notifications_dupes_backup AS SELECT …` (a copy — original rows untouched).
3. Only then remove them from the live table, **inside a transaction your team inspects and commits manually**.

This plan deliberately does **not** provide a delete script — that decision and that keystroke must stay with your team.

**Runtime queries are a different thing (and nothing new):** the notification service's claim query (`UPDATE notifications SET status='PROCESSING' …`) is something the *application* runs while operating — exactly the way today's code already runs `UPDATE … SET status='SENT'`. It touches only the `notifications.status` bookkeeping column, never user data, and your system already executes this kind of query every minute right now.

---

<a name="10-auth"></a>
## 10. Authentication

- **JWT verification is copied, never proxied.** The notification service gets copies of `authenticateUser.js` + `authorizeAccess.js` + cookie-parser, and verifies tokens with the same `JWT_SECRET`, read from environment variables. (The secret is currently hardcoded in the codebase — move it to env files in Phase 4 and **rotate it**, since it has lived in git.)

> **📘 New concept — "copied, never proxied":** each service checks the user's login token *itself*, using the same shared secret — it never forwards the request to another service to ask "is this user OK?". This keeps every request fast and means no service depends on another one just to answer "who is this?".

- **Sessions stay monolith-only.** The `express-session` file store (`./sessions` on local disk) never leaves the monolith. Before switch-over, verify no notification controller touches `req.session` (they appear to use JWT — confirm with a quick search).
- **CORS must match exactly.** The monolith currently allows `origin: true, credentials: true`. The notification service must copy this exactly at first — "almost the same" CORS is a classic source of *works in Postman, fails in the browser*. Tighten both to a real allowlist in Phase 4, together.
- **WebSocket endpoints stay open (no login)** — that is their contract today, and market prices are public data.
- **Internal endpoints** (detailed health, DLQ tools) bind to localhost/private network or require an internal header token — they are never reachable through nginx.

---

<a name="11-phases"></a>
## 11. The migration, phase by phase

Every phase ends with the system fully working. Every step lists its undo. **No step deletes code until the final phase.**

> **📘 New concept — feature flag (a "switch"):** a setting (usually an environment variable like `NOTIF_VIA_STREAM=true/false`) that turns a code path on or off **without redeploying**. Flags are how every risky step in this plan becomes instantly reversible.

### Phase 0 — Safety net (week 1). No behavior change.

```
WHAT PHASE 0 CHANGES  (answer: nothing that anyone can feel)

BEFORE:                                  AFTER (behavior 100% identical):

Admin Panel ────► :3002 monolith         Admin Panel ──► nginx ──► :3002 monolith
Mobile App  ────► :3002 monolith         Mobile App  ──► nginx ──► :3002 monolith
                                                          ▲
                                          nginx forwards 100% of traffic untouched.
                                          Think of it as installing a STEERING WHEEL
                                          now, which we only TURN in later phases.
```

| # | Action | Undo |
|---|---|---|
| 0.1 | **Deploy nginx routing 100% → monolith.** Point clients/DNS at it. Verify both WebSocket endpoints work *through* the proxy (watch the 30s-ping vs `proxy_read_timeout` interaction). | point DNS back at :3002 |
| 0.2 | **Record the golden files:** a full market day of `/ws/prices` and `/ws/market` messages; sample responses of every REST endpoint that will move; example `PRICE:LATEST:*` values and candle rows. | n/a |
| 0.3 | Add `/health` + `/ready` endpoints to the monolith. | remove them |
| 0.4 | Write integration tests for **all five notification paths** (§3.1 — especially trade auto-exit and plan-sync), plus tick delivery, subscribe/unsubscribe, and candle inserts. | n/a |
| 0.5 | **Settle the open questions:** is `smartapiStream`/order-depth alive anywhere (Fact 6)? Do any notification controllers use `req.session`? Which database holds `control_tradingdays`? Decide: delete or migrate. | n/a |

### Phase 1 — Seams inside the monolith (week 2). Still one process.

```
WHAT PHASE 1 CHANGES  (still ONE process — no new servers exist yet)

┌───────────────────────── monolith (:3002) ────────────────────────────┐
│                                                                       │
│  Trade hits target / plan changes / admin clicks "Send Now"           │
│        │                                                              │
│        ▼                                                              │
│  NotificationClient   ← new thin file, SAME function names as before  │
│        │                                                              │
│        ├─ switch OFF → calls the old functions directly (today's path)│
│        │                                                              │
│        └─ switch ON  → writes a message into the Redis Stream ────────┼──► Redis
│                                                                       │     │
│   in-process consumer reads the stream back ◄─────────────────────────┼─────┘
│        │                                                              │
│        ▼                                                              │
│   the SAME old send functions → Firebase → user's phone               │
└───────────────────────────────────────────────────────────────────────┘

WHY: we prove the "message pipe" works in real production for several days,
while everything still lives safely inside one process. If anything is off,
flip the switch back (seconds) and you are on today's exact path again.
```

| # | Action | Undo |
|---|---|---|
| 1.1 | Add `notificationClient.js`; switch the 4 import sites (§7.2). With `NOTIF_VIA_STREAM=false` it calls the old functions directly — **zero behavior change on day one**. | flip switch / revert imports |
| 1.2 | Turn the switch on: the monolith writes to `notif:dispatch` **and consumes its own stream in-process**, calling the same old functions. This proves the message format, the claim logic, retries, and the DLQ — with zero new processes and zero network hops. Run it for several days. | flip the switch |
| 1.3 | Database groundwork (run by hand, §9.5): duplicate-check, then the unique constraint; the two claim columns; deploy the claim-based cron logic (still inside the monolith). | drop constraint/columns |
| 1.4 | Compare all golden files — must be identical. | — |

### Phase 2 — Notification service extraction (weeks 3–4).

```
STEP 2.2 — move routes ONE SMALL GROUP at a time (the canary)

                        ┌──► :3005  notification service (new)
   clients ──► nginx ───┤
                        └──► :3002  monolith (everything else, unchanged)

   (a) first, ONLY  /api/notification-query .... → :3005   watch for 1 day
   (b) then add     /api/notifications (CRUD) .. → :3005   watch
   (c) then add     user-feed endpoints ........ → :3005   ← the MOBILE APP calls
                                                             these — watch error
                                                             rates extra closely
   (d) finally      /api/notification/send* .... → :3005

   Undo at ANY step = flip that one nginx line back. Seconds. No deploy.
```

```
STEP 2.3 — flip the consumer (who reads the message pipe)

BEFORE the flip:                          AFTER the flip:

monolith ──► [stream] ──► monolith's      monolith ──► [stream] ──► notification
                          own consumer                              service consumer
                              │                                          │
                              ▼                                          ▼
                          FCM push                                   FCM push

Exactly ONE consumer is active at a time (controlled by two switches).
And even if both accidentally ran together, the database claim (§7.4)
makes a double-send IMPOSSIBLE — that is lock-level safety, not switch-level hope.
```

| # | Action | Undo |
|---|---|---|
| 2.1 | Start `notification-service` on :3005 (moved code, own pools, copied auth/CORS, Firebase JSON, IST timezone). Stream consumer **off**, cron **off**, REST **on**. | stop the process |
| 2.2 | **Canary by route group via nginx**, lowest-risk first (diagram above). | flip the nginx line back (seconds) |
| 2.3 | **Flip the consumer:** monolith's in-process consumer off, the service's consumer on. Trade auto-exit, expiry, and plan-sync pushes now flow monolith → pipe → service → FCM. Re-run the Path 3/4/5 tests from Phase 0.4. | flip both switches back |
| 2.4 | **Flip the cron** to the service. A brief overlap is harmless thanks to the database claim. | flip back |
| 2.5 | **Bake for 2 weeks.** The monolith's notification code sleeps behind switches but stays deployable in minutes. Then — and only then — delete it and the monolith's Firebase JSON. | (after deletion) git revert |

### Phase 3 — Live data extraction (weeks 5–7). The hard one — done without ever dual-running.

```
STEP 3.1 — test the new WebSocket layer on REAL live data, with ZERO user risk

   BROKERS ──► monolith (:3002 — still owns ALL ticks, exactly as today)
                   │
                   ▼  writes PRICE:LATEST:{symbol}
              [ shared Redis ] ◄──────────────────┐
                                                  │ reads price snapshots
   test client ──► live-data service (:3004) ─────┘
                   (brokers switched OFF — it ONLY answers
                    WebSocket SUBSCRIBE requests from Redis)

   Production users never touch :3004 yet.
   If the new service is broken, nobody notices except us.
```

```
STEP 3.4 — the switch-over, on a weekend evening
            (market CLOSED  =  zero live ticks anywhere  =  nothing can be missed)

   ①  monolith:   BROKERS_ENABLED=false  → restart
        └── the monolith stops talking to Angel One / Dhan completely
   ②  live-data:  BROKERS_ENABLED=true
        └── the live-data service logs in to Angel One —
            it is now the ONLY process anywhere that logs in
   ③  nginx:      /ws/prices + /ws/market  → :3004     (one reload, sub-second)
   ④  monolith:   LIVE_DATA_EXTRACTED=true
        └── the Angel token-reader and broker-status proxy switch on

   UNDO = do ④ ③ ② ① in reverse. Under 5 minutes,
   still inside the closed window — users never see a thing.
   (And because nothing ever dual-runs, the undo has no session conflicts either.)
```

| # | Action | Undo |
|---|---|---|
| 3.1 | Start `live-data-service` on :3004 with the full move list (§8.1) and **brokers disabled by switch**. It serves both WebSocket endpoints, answering SUBSCRIBE with Redis snapshots — which still come from the monolith's ticks. The entire WebSocket/subscription layer gets validated on **live production data** with zero user exposure (diagram above). | stop the process |
| 3.2 | **Replay validation:** feed the recorded golden ticks through the service's full pipeline (TickGate → aggregator → broadcast) in a test environment. Compare the produced candle rows and WebSocket messages to the golden files. | n/a |
| 3.3 | Ship the two monolith-side adapters **dark** (present but inactive until the flag flips): the Angel token-reader (§8.5) and the broker-status proxy (§8.6). | flag stays off |
| 3.4 | **Switch over in a market-closed window** (diagram above). The Angel login, 8:45 cron, and token scheduler now run **only** in the live-data service. | reverse ①–④, < 5 min |
| 3.5 | **War-room the next market open** with the checklist in §15.2: brokers connect on schedule, ticks flow, Redis keys fresh, candles being written, charts moving, clients reconnected. | switches + nginx back; brokers restart in the monolith in < 5 min |
| 3.6 | **Bake 2–4 weeks** with the monolith's socket/broker code dormant. Then delete `sockets/`, `core/`, `brokers/`, and the moved `marketData/` files from the monolith. | git revert |

### Phase 4 — Hardening (week 8).

Secrets into env files / a secret manager (+ rotate the two hardcoded secrets), the read-only targeting role live (§9.4), a real CORS allowlist everywhere, remove the now-unneeded switches, dashboards and alerts live (§13), delete `smartapiStream.js` if Phase 0.5 confirmed it dead, update documentation.

---

<a name="12-risks"></a>
## 12. Risk register

| # | Risk | Phase | Chance | Impact | Protection |
|---|---|---|---|---|---|
| 1 | Trade auto-exit / plan-sync pushes silently lost (Facts 1–2) | 2 | Certain *if the client is skipped* | High | `NotificationClient` + dedicated tests for paths 3/4/5 + FCM failure-rate alarm |
| 2 | Candle writing stops (Fact 3) | 3 | Certain *if the aggregator is left behind or made read-only* | High | aggregator moves with a write pool; candle-insert alarm |
| 3 | Monolith fails to boot on a missing import (Fact 4) | 3 | Certain *if the token-reader is skipped* | High but loud | token-reader shipped dark before switch-over |
| 4 | Angel One session war (two logins fighting) | 3 | High if anything dual-runs | High | the single-owner rule; nothing ever dual-runs |
| 5 | Mobile app breaks on a moved URL | 2 | High without a gateway | High | nginx keeps every public URL identical forever |
| 6 | Duplicate notifications (double cron / redelivery) | 2+ | Medium | Medium | database claim + unique constraint — duplicates impossible by construction |
| 7 | nginx cuts idle WebSocket connections | 0 | Medium | Medium | `proxy_read_timeout 120s` > the 30s ping; verified in Phase 0.1 |
| 8 | Notification service down → triggers lost | 2+ | Low | Medium | the stream holds messages durably; backlog alarm; processed on recovery |
| 9 | Reconnect storm at WebSocket switch-over | 3 | Medium | Low | switch-over happens in a closed window; clients already auto-reconnect |
| 10 | Wrong timezone on the new service's machine | 2 | Medium | Medium | explicit TZ check in the deploy script + a scheduled-notification smoke test |
| 11 | `module-alias` paths break when files move | 2–3 | Medium | Low, loud | a boot smoke test per service in CI |
| 12 | Database connection exhaustion | 2–3 | Low | Medium | the pool budget (§9.2) + pool alarms |
| 13 | Order-depth "breaks" — but was already dead (Fact 6) | 3 | Medium | Low | Phase 0.5 settles its status before anyone blames the migration |

---

<a name="13-observability"></a>
## 13. Monitoring and alerts

> **📘 New concept — correlation ID:** a unique label stamped on a request when it first arrives, then carried along everywhere that request goes — through the monolith, through the message pipe, into the notification service, onto the Firebase call. Searching the logs for that one ID shows the request's entire journey across all services. It is how you debug a multi-service system without going crazy.

nginx stamps `X-Correlation-Id` on every request; services log it; the monolith puts it into every stream message (§7.2); the notification service logs it on consume and on the FCM send. One search then traces *admin click → pipe → Firebase message ID*.

**Alarms that specifically catch this plan's failure modes:**

| What we measure | Service | Alarm when | Catches |
|---|---|---|---|
| tick end-to-end delay (exchange timestamp → send to client) | live-data | p99 > 2s for 1 min (market hours) | slow pipeline |
| ticks per second, per broker | live-data | < 1 for 30s during market hours | dead broker connection |
| broker connection state | live-data | disconnected > 60s in market hours | session/login trouble (risk 4) |
| **candle insert rate** | live-data | 0 inserts for 3 min in market hours | **the Fact-3 failure class** |
| TickGate rejection % | live-data | > 50% for 5 min | bad upstream data |
| WebSocket client count | live-data | 0 for 2 min in market hours | proxy/switch-over mistakes |
| message-pipe backlog (pending + lag) | notif | backlog > 100 or lag > 60s | stuck or dead consumer |
| DLQ size | notif | > 0 | poisoned messages |
| FCM failure rate | notif | > 10% over 5 min | **the silent-loss failure class** |
| cron last-claim time | notif | > 3 min old | dead scheduler |
| DB pool idle/waiting | all | 0 idle for 30s | risk 12 |
| `PRICE:LATEST:*` freshness, sampled **from the monolith side** | monolith | older than 90s in market hours | end-to-end check from the *consumer's* viewpoint |

**Logging:** structured JSON in production, one shared format. The live-data service logs state changes and rejections only (never each tick — far too noisy). The notification service logs every send attempt with its correlation ID and Firebase message ID.

---

<a name="14-testing"></a>
## 14. Testing strategy

1. **Golden-file contract tests** (built in Phase 0.2, run at every later step): recorded WebSocket messages, REST response shapes, Redis value shapes, candle rows — compared byte-for-byte where outputs are deterministic, schema-checked where they are not.
2. **The five-path notification matrix** — each path in §3.1 gets an end-to-end test: trigger → (pipe) → `push_notifications` row → FCM call (faked in CI, one real device in staging). Paths 3, 4 and 5 are the tests that protect the hidden connections in Facts 1–2.
3. **Duplicate-protection tests:** deliver the same pipe message twice → exactly one push; run two crons at once → exactly one claim; kill the consumer mid-batch → redelivery → still exactly one push.
4. **Replay tests** for live data (Phase 3.2): golden ticks in → identical candles and messages out.
5. **Chaos drills in staging:** kill the notification service for 10 minutes during sends (messages wait in the pipe, drain on restart); kill the live-data service (clients reconnect, monolith REST unaffected, brokers reconnect by their built-in state machine); restart Redis (services reconnect; the stream group survives **if Redis persistence is on — check the `appendonly`/RDB config**, because message durability now depends on it).
6. **A full dress rehearsal of the Phase 3.4 switch-over** in staging, with a stopwatch, before doing it in production.

---

<a name="15-runbooks"></a>
## 15. Cutover and rollback runbooks

### 15.1 Live-data switch-over (market-closed window)

```
[ ] Market is closed and the next open is > 12h away
[ ] Staging dress rehearsal passed within the last week
[ ] Golden files current; replay test green
[ ] Monolith: BROKERS_ENABLED=false → restart → confirm zero broker logins in its logs
[ ] Live-data: BROKERS_ENABLED=true → Angel login succeeds ONCE, token visible in ANGEL:SESSION
[ ] nginx: flip /ws/prices + /ws/market upstreams → reload → test client connects,
    snapshot replies arrive
[ ] Monolith: LIVE_DATA_EXTRACTED=true (token-reader + broker proxy now active)
[ ] Admin panel loads; charts/history/movers serve (stale-but-served is CORRECT while closed)
[ ] War-room scheduled for the next market open
```

### 15.2 First-market-open war room

```
[ ] Market scheduler flips to OPEN; brokers auto-connect (state visible in logs)
[ ] ticks/sec > 0 within 60s of open;  end-to-end tick delay p99 < 2s
[ ] PRICE:LATEST freshness < 90s — checked FROM the monolith (the consumer's viewpoint)
[ ] Candle inserts appearing in TimescaleDB        ← the Fact-3 check
[ ] Admin panel: live prices moving; charts updating; movers fresh
[ ] broker-status page shows real broker health (the proxy works)
[ ] ZERO Angel auth errors in EITHER service's logs (the single-owner check)
[ ] WebSocket client count ≈ the pre-migration normal
```

### 15.3 Rollback, per phase

| Phase | Action | Time |
|---|---|---|
| 0–1 | flip one switch (`NOTIF_VIA_STREAM=false`) | seconds |
| 2 (routes) | flip the nginx upstream line, reload | seconds |
| 2 (consumer/cron) | flip two switches; the DB claim makes the overlap safe | < 1 min |
| 3 | reverse the four switch-over steps; brokers restart in the monolith | < 5 min, no session conflicts because nothing ever dual-ran |
| after deletion | git revert of the deletion commit + redeploy | < 30 min — which is exactly why deletion waits for the bake period |

---

<a name="16-structure"></a>
## 16. Folder structure and deployment

### 16.1 Target layout

```
trade-platform/
├── gateway/
│   └── nginx.conf                  ← the routing table IS the migration state;
│                                      it must live in version control
├── main-backend/                   ← the monolith (slimmed down at the end)
│   └── src/
│       ├── index.js                ← no socket/broker/notification code (after bake)
│       ├── controllers/  routes/  middleware/  models/  db/  utils/
│       └── services/
│           ├── notificationClient.js        ← NEW (tiny, stays forever)
│           └── marketData/angelTokenReader.js ← NEW (tiny, stays forever)
│
├── notification-service/
│   └── src/
│       ├── server.js               ← Express + cron startup
│       ├── config/      (env, firebase, logger)
│       ├── middleware/  (authenticateUser, authorizeAccess — copies)
│       ├── services/    (push, notificationBroadcast, autoNotification, cron)
│       ├── controllers/ (the three notification controllers)
│       ├── routes/      (the three notification route files)
│       ├── consumers/   (dispatchConsumer.js, dlqRedrive.js)
│       └── db/          (db.js — victory_db pool max 5,
│                          targetingDb.js — the read-only role, §9.4)
│
├── live-data-service/
│   └── src/
│       ├── server.js               ← HTTP + WebSocket startup, /health
│       ├── config/      (env, brokerConfig, logger)
│       ├── sockets/     (all 8 files, as-is)
│       ├── core/        (all 6 files, as-is)
│       ├── brokers/dhan/ (all 4 files, as-is)
│       ├── services/marketData/ (angel auth/login/scheduler/WS v1+v2,
│       │                          realtimeAggregator ← the candle writer)
│       └── db/          (redisClient, timescaleClient — READ/WRITE pool max 10)
│
└── strategy-engine/                ← untouched
```

### 16.2 Deployment, honestly

Today this stack runs as plain Node processes on a Windows/EC2 machine — not in Docker. Containerizing is a fine future goal, **but doing two migrations at once is how both fail**. So: run the new services the same way the current one runs, managed by PM2.

> **📘 New concept — PM2:** a production process manager for Node.js. It starts your apps, restarts them automatically if they crash, captures their logs, and survives reboots. It is the simplest professional upgrade from "nodemon in a terminal window".

```js
// ecosystem.config.js
module.exports = {
  apps: [
    { name: "monolith",      cwd: "./main-backend",         script: "src/index.js",  env: { PORT: 3002 } },
    { name: "notif-svc",     cwd: "./notification-service", script: "src/server.js", env: { PORT: 3005, TZ: "Asia/Kolkata" } },
    { name: "live-data-svc", cwd: "./live-data-service",    script: "src/server.js", env: { PORT: 3004 },
      max_memory_restart: "1G" },
  ],
};
```

Move to Docker/Compose as a separate project later, once the service boundaries have proven themselves stable.

---

<a name="17-bites"></a>
## 17. Things that will bite you — the checklist of classic mistakes

1. **Forgetting cookie-parser in the notification service** — JWT arrives in a cookie; without the parser, every authenticated request fails with "no token".
2. **Mixing Redis client libraries** — this codebase uses the `redis` npm package; the Strategy Engine uses `ioredis`. Pick one per service and stay consistent — their APIs differ subtly.
3. **CORS not byte-identical** — copy the monolith's CORS settings exactly at first; tighten later, everywhere at once (§10).
4. **Firebase service-account JSON path** — it's referenced by path in `config/firebase.js`; the path must be updated for the new service's directory.
5. **MarketScheduler running in two places** — only the live-data service may run it after Phase 3, or brokers get double connect/disconnect commands.
6. **BrokerManager is a singleton** — in the new process it initializes fresh; make sure every broker config is passed in explicitly.
7. **`InternalTokenList` polling** — the WebSocket layer refreshes its enabled-symbols list from the DB every 30s; the live-data service needs DB access for this from day one.
8. **The `broadcastPrice` callback chain** — in `index.js`, `broadcastPrice` is handed into the Angel WebSocket setup as a callback. The whole chain moves together or ticks vanish.
9. **Graceful shutdown** — the live-data service needs its own SIGTERM handler calling `BrokerManager.stopAll()`, like the monolith has today.
10. **Timezone** — the scheduled-notification cron compares IST times. The new machine must be on Asia/Kolkata (or the code's IST conversion verified) — see risk 10.
11. **`angel.service.js:2` imports a file you are moving** (Fact 4) — ship the token-reader *before* switch-over day, or the monolith won't boot.
12. **The four import-swap sites** (§7.2) — after swapping, search the monolith for `notificationBroadcast.service`, `planSync.service`, and `config/firebase`; the only remaining hit should be `notificationClient.js` itself.
13. **`push_notifications` may already contain duplicates** — the unique constraint won't apply until they're handled. Check early (Phase 1.3), and follow the archive-first procedure in §9.5 — never a bare `DELETE` on production rows.
14. **nginx `proxy_read_timeout` vs the 30s WebSocket ping** — the default 60s is uncomfortably close; set 120s+ or you will chase phantom disconnects.
15. **`module-alias` in the monolith's package.json** — moved files that used alias paths resolve differently in a new repo. A boot smoke test per service in CI catches this in seconds.
16. **Redis persistence** — notification delivery durability now depends on Redis surviving a restart. Confirm AOF or RDB persistence is enabled; default Windows Redis 5 configs are often weak here.
17. **`client_max_body_size` in nginx** — the monolith accepts 50 MB JSON; nginx defaults to 1 MB and will reject the market-data imports the day after you add the proxy. Set it in Phase 0.1.
18. **`uploads/` and `charting_library-master`** are served from the monolith's local disk — they stay with the monolith; don't route those paths anywhere else.
19. **`broker.controller.js`** must flip to the proxy **in the same deploy** as the live-data switch-over (Fact 5) — not "later", or the broker status page lies in between.
20. **Don't "improve" code mid-move.** Every refactor bundled into the migration multiplies what you must reason about when something looks wrong at 9:15 AM on the first market open. Move ugly code as-is; schedule the beautification for after the bake period.

---

<a name="18-order"></a>
## 18. Final implementation order — one page

```
Week 1    Phase 0: nginx pass-through · golden files recorded · health checks
          · five-path notification tests · open questions settled
          (smartapiStream alive? req.session used? control_tradingdays location?)

Week 2    Phase 1: NotificationClient + 4 import swaps · message pipe running
          INSIDE the monolith (switch-controlled) · claim columns + unique
          constraint (run by hand, after backup)

Weeks 3–4 Phase 2: notification service up · nginx canary, one route group at a
          time (query → CRUD → user feed → send) · consumer flip · cron flip
          · 2-week bake → only then delete the monolith's notification code

Weeks 5–7 Phase 3: live-data service up (brokers OFF — WebSocket layer validated
          on live Redis data) · golden replay test · token-reader + broker proxy
          shipped dark · weekend market-closed switch-over · war-room the first
          market open · 2–4-week bake → only then delete the monolith's
          socket/broker code

Week 8    Phase 4: secrets rotated into env · read-only targeting role ·
          CORS allowlist · alarms live · documentation updated
```

**What we deliberately do NOT touch:** the Strategy Engine (:3003) and its signal WebSocket — it is already a separate, working service; the data pipeline and its `ohlc:candle_close` Redis events — already decoupled; JWT issuance — unchanged; database schemas — unchanged except the two additive notification columns/constraint; and any business logic beyond the explicitly listed import swaps and the two small adapters.
