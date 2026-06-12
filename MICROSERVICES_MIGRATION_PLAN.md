# Victory Platform — SaaS-Grade Microservice Extraction Plan
## Notification Service + Live Data Service (Zero Functionality Break)

**Status:** Authoritative. Supersedes the Gemini-generated `migration_architecture_plan.md`.
**Date:** 2026-06-12
**Verification:** Every defect and every coupling claimed in this document was verified by reading the actual source code in `D:\Backend\new_backend\src`. File and line references are given everywhere so you can check them yourself.

---

## Table of Contents

1. [TL;DR — the plan in five sentences + the five safety locks](#1-tldr)
2. [Verdict on the Gemini plan — what it got right, and the 5 places it would have broken production](#2-verdict)
3. [The corrected system map (what really calls what)](#3-system-map)
4. [Target architecture](#4-target-architecture)
5. [The gateway layer (the keystone Gemini was missing)](#5-gateway)
6. [Notification Service — corrected design](#6-notification-service)
7. [Live Data Service — corrected design](#7-live-data-service)
8. [Database, Redis, and "who owns what" rules](#8-data-ownership)
9. [Authentication](#9-auth)
10. [The phased migration plan (zero downtime, realistic rollback)](#10-phases)
11. [Risk register](#11-risks)
12. [Observability](#12-observability)
13. [Testing strategy](#13-testing)
14. [Cutover and rollback runbooks](#14-runbooks)
15. [Folder structure and deployment](#15-structure)
16. [Things that will bite you (corrected and extended list)](#16-bites)

---

<a name="1-tldr"></a>
## 1. TL;DR — the plan in five sentences

1. **First put a reverse proxy (nginx) in front of everything** so all clients — the admin panel *and the mobile app* — keep talking to one address forever; after that, moving a route to a new service is a one-line config change, and rolling back is flipping that line back.
2. **Extract the Notification Service first** (lower risk), but — unlike the Gemini plan — the monolith keeps a thin, drop-in `NotificationClient` because **three business flows inside the monolith fire notifications in-process** (trade auto-exit, trade expiry cron, plan-sync pushes) and would silently die otherwise.
3. **Extract the Live Data Service second**, moving the *entire* tick path **including `realtimeAggregator` with a WRITE database pool** (the Gemini plan made it read-only, which would have silently stopped all candle/chart data), and making the live-data service the *single owner* of the Angel One login while the monolith becomes a token *reader*.
4. **Never run two copies of the broker connections at the same time** (the Gemini "dual-run" plan would make the two processes invalidate each other's Angel One session); instead, cut over during a market-closed window, when there are zero live ticks and therefore zero user impact.
5. **Never delete monolith code on cutover day** — keep the old path dormant behind a flag for a 2–4 week bake period, then delete.

### 1.1 The five safety locks — how we make sure nothing breaks

First, an honest sentence: no engineer on earth can promise a literal "100%" — anyone who does is selling something. What we **can** do — and what this plan does — is design the migration so that:

- every risky step is **checked against recorded reality** before real users ever touch it, **and**
- every step can be **undone in seconds to minutes**, **and**
- any problem that somehow slips through is **caught by an alarm, not by a customer**.

When all three are true at every step, the practical risk of losing functionality is as close to zero as production engineering allows. This plan achieves it with **five independent locks** — even if one lock somehow fails, the other four still protect you. This is the same pattern (proxy → flags → canary → golden tests → bake period) that large SaaS companies use for exactly this kind of extraction.

**🔒 Lock 1 — Clients never change, ever.**
The web admin panel and the mobile app keep talking to one single address forever. All the "moving" happens behind the nginx curtain. A client that doesn't change cannot break.

**🔒 Lock 2 — Every step has a seconds-level undo.**

```
  THE ROLLBACK LADDER — how fast can we undo, at any moment?

  Phase 0   nginx pass-through installed   →  point DNS back............(minutes)
  Phase 1   message pipe inside monolith   →  flip 1 flag...............(seconds)
  Phase 2   notif routes moved to :3005    →  flip 1 nginx line.........(seconds)
  Phase 2   consumer + cron flipped        →  flip 2 flags..............(< 1 min)
  Phase 3   live-data cutover              →  4 steps in reverse........(< 5 min)
  Bake      is old code deleted? NO        →  old path redeployable.....(minutes)
```

There is **no point in this entire migration** where going back takes longer than a coffee break.

**🔒 Lock 3 — We compare against recorded reality, not against hope.**

```
  Phase 0: RECORD what the system               Later: COMPARE the new service
  does today (the "golden files")               against those recordings

  ┌──────────────────────┐                      ┌────────────────────────────┐
  │  live system today   │                      │  new service (in staging)  │
  │   WS messages ───────┼──► saved files  ───► │   same inputs replayed     │
  │   REST responses ────┼──► saved files       │   outputs diffed vs files  │
  │   Redis values ──────┼──► saved files       │                            │
  │   candle DB rows ────┼──► saved files       │   ANY difference =         │
  └──────────────────────┘                      │   STOP → fix → re-test     │
                                                └────────────────────────────┘
```

"Nothing changed" stops being an opinion and becomes a **byte-for-byte verified fact**.

**🔒 Lock 4 — Only one thing changes at a time.**
Every phase moves exactly one part while everything else stays frozen. If anything misbehaves, there is exactly **one suspect** — no guessing, no debugging two changes at once at 9:15 AM.

**🔒 Lock 5 — Old code is never deleted until the new path has survived weeks of real production.**
During the bake period the monolith can take back any job within minutes. Deletion happens only after the new service has handled real market days and real notification traffic without incident.

### 1.2 What this plan will NEVER do (hard rules)

1. **No AI tool runs any database query — ever.** Nothing in this migration has been executed against your database, and nothing will be. The only SQL in this entire plan is **three additive statements** (§8.5) that **your team runs by hand**, after taking a backup.
2. **No existing data is modified or deleted by the migration.** No `DROP`, no `DELETE`, no `UPDATE` of user data — the three statements only *add* (two columns, one constraint, one read-only login).
3. **No URL, payload shape, event name, or WebSocket message ever changes.** The contracts in §7.2 are frozen and verified by golden tests.
4. **No two broker connections at the same time** — that is the thing that would corrupt Angel One sessions (§2.3 Error B).
5. **No business-logic refactoring during the move.** Code moves as-is. Only the explicitly listed import swaps (§6.2) and two small adapters (§7.4, §7.5) change — and each is named, file by file, line by line.

---

<a name="2-verdict"></a>
## 2. Verdict on the Gemini plan

### 2.1 What it got right (keep these decisions)

The plan is not garbage — its file inventory is largely accurate, and several of its judgment calls are correct. We keep them:

| Decision | Verdict |
|---|---|
| Do **not** extract the Signal WebSocket from the Strategy Engine (:3003) | ✅ Correct. It's ~170 lines of same-process unicast. Extracting it adds latency and race conditions for zero benefit. |
| Use **Redis Streams** (not Pub/Sub, not RabbitMQ/Kafka) for notification dispatch | ✅ Correct choice. Durable, replayable, no new infrastructure. |
| Keep a **shared database** for now; don't split schemas | ✅ Correct. Splitting DBs during extraction multiplies risk. |
| Don't touch the data pipeline / `ohlc:candle_close` flow | ✅ Correct. Already decoupled via Redis. |
| Extract notifications **before** live data | ✅ Correct ordering. |
| Move code **as-is**, don't refactor business logic during the move | ✅ Correct principle (the plan then violates it by missing required changes — see below). |

### 2.2 The five verified defects that would have broken production

Each defect below follows the same pattern: **what the plan says → what the code actually does → what breaks**. Every claim has a file:line reference you can open.

---

#### Defect 1 — "Move all notification code out, then delete it from the monolith" silently kills trade auto-exit notifications

**The plan says** (its §4.3, §10 Phase 2.2/2.11): move `notificationBroadcast.service.js` and friends entirely to the new service, then remove notification code from the monolith.

**The code actually does:** the monolith's own business logic calls the notification service *as an in-process function*, from places the plan never lists:

- `src/controllers/tradeRecommendations.controller.js:6-7` imports `broadcastNotificationByQuery` directly.
- It calls it at lines **939, 956, 973, 989** inside `sendAutoExitNotifications()` — this is what notifies users when a trade recommendation hits **target, stop-loss, or expiry**.
- That function is invoked from at least four call sites (lines **1218, 1397, 1451, 1502**), including the expiry path.
- `src/services/tradeExpireCron.js:2` requires `autoCloseExpiredTrades` from that same controller — so the **cron that stays in the monolith** drives notifications through the **code that was just deleted**.

**What breaks:** after Phase 2.11, every auto-exit / stop-loss / target-hit / expiry push notification stops. And here is the nasty part — every one of these calls is wrapped in `try/catch` with only a `console.error` (e.g. line 1224: *"Failed to send auto-close notification"*). The app does not crash. Nothing alerts. **Notifications just silently stop arriving**, and you find out from angry users.

---

#### Defect 2 — `planSync.service.js` is a second, completely unlisted Firebase consumer in the monolith

**The plan says:** the only Firebase consumers are the notification services and controllers it lists. Remove the Firebase service-account JSON from the monolith after extraction (its §14.4 point 3).

**The code actually does:**

- `src/services/planSync.service.js:2` — `const admin = require("../config/firebase")` — it talks to Firebase **directly**, sending *silent data-only pushes* (line 40-48) that tell the mobile app a user's plan changed.
- It is called from billing flows that stay in the monolith: `src/controllers/plansNew.controller.js:74` and `src/controllers/proPlanTransactions.controller.js:90` and `:155`.

**What breaks:** the moment Firebase credentials leave the monolith, **plan purchases and plan changes stop syncing to users' phones**. Again wrapped in `catch → console.error` (lines 65, 113) — a silent failure in the *billing* path, the worst possible place.

---

#### Defect 3 — "Live data service gets a read-only DB pool (max 3)" silently stops all candle/chart data

**The plan says** (its §8.1, §10 Phase 3.3): the live-data service accesses `victory_market_db` **read-only**, pool max 3, only for `InternalTokenList` and `control_tradingdays`.

**The code actually does:**

- `src/services/marketData/realtimeAggregator.service.js` is required by the Angel One tick path itself — `angelWebSocket.service.js:6` and `angelWebSocket.service.v2.js:7`.
- The aggregator **builds 1-minute OHLC candles from live ticks and WRITES them to TimescaleDB**: it imports the Timescale client (line 2), runs **"🔥 Parallel DB inserts"** (line 54), calls `newOhlcQueries.insert1MinCandle(data)` (line 218) and `timescaleClient.query(...)` (lines 222-227) on a flush timer (line 25).
- The aggregator does not even appear in the plan's move list or folder structure.

**What breaks:** move the brokers out with a read-only pool and **every candle insert fails**. Charts, history, intervals, movers, sparklines — everything that reads `OHLC_*` tables — slowly goes stale during market hours. Live tick prices would still flash on screen (making it *look* fine in a quick test), while the underlying historical data quietly stops being written. This is precisely the kind of "features which are working right now smoothly will not work smoothly" failure your senior reviewers warned about.

**Correction:** `realtimeAggregator.service.js` (and its write path) moves **with** the tick path, and the live-data service gets a **read/write** pool sized for parallel inserts (start at 10, tune from pool metrics).

---

#### Defect 4 — Moving `marketData/angelAuth.service.js` crashes the monolith on boot

**The plan says** (its §12.1): all of `services/marketData/angel*` move to the live-data service.

**The code actually does:**

- `src/services/angel.service.js:2` — a service that **stays in the monolith** (it serves historical-candle REST data; see the SmartAPI Historical docs comment at line 19) — does `require('./marketData/angelAuth.service.js')`.

**What breaks:** after the file move, the monolith's `require()` resolves to a missing file and the **whole monolith fails to start**. This one at least fails loudly — but it would have failed *on cutover day*, in production, at the worst time.

**Correction:** the monolith keeps a small *token-reader* module (see §7.4), and `angel.service.js` switches one import. The Angel One *login/refresh* moves to the live-data service as the single owner.

---

#### Defect 5 — `broker.controller.js` reads the in-process BrokerManager singleton and would silently lie after extraction

**The plan says:** nothing. This file is never mentioned.

**The code actually does:** `src/controllers/broker.controller.js:1,5` calls `getBrokerManager()` — the in-process singleton. This serves the admin panel's broker status endpoint.

**What breaks:** after extraction, `getBrokerManager()` in the monolith returns a **fresh, empty singleton** (no registered brokers). The endpoint doesn't error — it reports "no brokers", which is worse than an error because it looks like a real outage.

**Correction:** `broker.controller.js` becomes a thin HTTP proxy to the live-data service's `/health` (see §7.5).

---

### 2.3 Four serious design errors (beyond outright breakage)

#### Error A — "Just change the frontend URL to :3004/:3005" ignores the mobile app, and the anti-gateway argument is wrong

The plan's only client-side change is editing URLs in the React admin panel. But this platform has a **mobile app** (the entire FCM/`push_devices` pipeline exists to push to phones; endpoints like `getNotificationUserWise`, `mark-delivered`, `mark-opened` are user-feed endpoints called by the app, not the admin panel). **You cannot hot-edit a URL inside apps already installed on users' phones.** Any plan that moves public endpoints to new ports breaks shipped clients or forces a coordinated app release + forced upgrade.

The plan explicitly rejects a gateway for WebSockets, claiming "API gateways add latency to every tick (microseconds matter for market data)". This is not a real concern: an nginx proxy hop inside the same machine/VPC adds **well under a millisecond**, while the tick already travels tens of milliseconds over the public internet to the browser. nginx proxies WebSockets natively and handles millions of concurrent connections in production at companies far larger than this one.

**Correction:** a reverse proxy is the *first* thing we deploy (§5). After that, no client — web or mobile — ever changes a URL, and every cutover/rollback becomes a one-line routing change.

#### Error B — "Dual-run" of broker connections (its Phase 3.6–3.8) is built on sand

The plan's centerpiece for live-data cutover is running broker connections in the monolith **and** the new service simultaneously, then comparing output. Three reasons this fails:

1. **Session invalidation.** Both processes would run `AngelAuthService.login()` / the token scheduler / the 8:45 AM cron with the *same system credentials*. Broker session APIs commonly invalidate the previous session on a new login — the two processes would knock each other offline in a loop. The plan even admits "some brokers may reject the second connection" (its §14.4) and proceeds anyway.
2. **The dedup claim is false.** The plan says TickGate's 50ms duplicate suppression "prevents broadcast duplicates" during dual-run. TickGate state is **per-process memory** — two processes each have their own TickGate and suppress nothing across each other.
3. **The comparison is meaningless.** Two independent WS connections receive ticks at slightly different times; `PRICE:LATEST` last-writer-wins flapping between two writers makes "100% parity" unmeasurable.

**Correction:** no dual-run, ever. Cut over in a **market-closed window** (evening/weekend) when there are zero live ticks, validate the new service's pipeline against **recorded golden traffic** beforehand, and war-room the next market open (§10 Phase 3, §14).

#### Error C — Idempotency by feature flag is not idempotency

"Only ONE cron runs, controlled by feature flag" is a *hope*, not a guarantee. Deploy overlaps, crash-restarts, or a mistyped flag means two crons. And even today, the code marks a notification `SENT` *after* pushing — a crash between push and mark means a duplicate blast on restart.

**Correction:** make duplicates *impossible at the database level*, not the process level — an atomic claim (`UPDATE … WHERE status='PENDING' RETURNING`) before sending, and a unique constraint on `push_notifications(notification_id, user_id)`. Then a double cron is harmless by construction (§6.4).

#### Error D — It plans to migrate dead code through the riskiest phase

`src/services/smartapiStream.js` (the order-depth feed) is **not required by any file in `src/`** — the only references are comments in `Migration/028_fix_on_conflict_constraints.sql`. It is never started from `index.js`. The plan treats it as a live component, moves it, and budgets risk for it. Meanwhile `indexCalculation.service.js` reads `Order_Depth:*` Redis keys that this dormant file was supposed to write — **verify whether the depth feature works at all today** before spending any migration effort on it (§10 Phase 0.5).

### 2.4 Smaller gaps (fixed in this plan)

- **No horizontal-scaling path for the WS service** — ingest (broker connections) and fan-out (client WebSockets) stay fused in one process with in-memory callbacks, so you can never run two instances. We add an internal Redis pub/sub seam now, cheaply (§7.3).
- **No backpressure handling** — broadcasting with `ws.send()` to a slow client grows `bufferedAmount` unbounded (memory blowup). One guard line fixes it (§7.6).
- **Security regressions waved through** — `notification_query` executes **arbitrary SQL stored in a database table** using a full-privilege pool. Extraction is exactly the moment to run those queries under a read-only role with a statement timeout (§8.4). Also: hardcoded `JWT_SECRET`, hardcoded session secret, `cors({origin:true, credentials:true})`.
- **Deleting monolith code immediately after cutover** — we keep dormant flagged paths through a bake period instead (§10).
- **No end-to-end tick-lag metric, no Stream backlog alerting, no DLQ** (§12).
- **An accidental benefit never mentioned:** today, `process.on("uncaughtException", … process.exit(1))` in `index.js:464` means **any crash in any of ~150 admin routes kills the live market feed for every connected user**. Extraction fixes this blast-radius problem — it's one of the strongest arguments *for* this migration and belongs in the business case.

---

<a name="3-system-map"></a>
## 3. The corrected system map

### 3.0 The system today, in one picture

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
    live market feed for every user, because index.js does process.exit(1)
    on any uncaught exception. Separating the services FIXES this —
    it is one of the strongest reasons to do this migration at all.
```

### 3.1 The five notification trigger paths (the plan knew about two)

```
PATH 1  Admin "Send Now"        → POST /api/notification/send → broadcastNotificationByQuery → FCM
PATH 2  Scheduled                → autoNotificationCron (every minute) → sendScheduledNotification → FCM
PATH 3  Trade auto-exit          → tradeRecommendations.controller (target/SL hit during LTP update,
        (MISSED BY GEMINI)         lines 1218/1397/1451) → broadcastNotificationByQuery → FCM
PATH 4  Trade expiry             → tradeExpireCron → autoCloseExpiredTrades →
        (MISSED BY GEMINI)         sendAutoExitNotifications (line 1502) → broadcastNotificationByQuery → FCM
PATH 5  Plan change (billing)    → plansNew.controller:74 / proPlanTransactions.controller:90,155
        (MISSED BY GEMINI)         → planSync.service → firebase-admin directly (silent data push)
```

Paths 3, 4, 5 originate inside monolith business logic. They are **why the monolith must keep a notification client** after extraction (§6.2).

### 3.2 The corrected live-data flow (with the part Gemini missed in bold)

```
BROKER (Angel One / Dhan)
   ↓ ws connection
Broker adapter (BaseBrokerWS subclass / angelWebSocket.service.v2)
   ↓ normalized tick
   ├─→ BrokerManager._onTick → TickGate (8 rules) → Redis SET PRICE:LATEST:{symbol} (60s TTL)
   │                                              → InternalMarketWS.pushPrice → /ws/market clients
   ├─→ broadcastPrice → ALL /ws/prices clients (admin panel marketWs.js)
   └─→ **realtimeAggregator: builds 1-min OHLC candles in memory,
        flush timer → PARALLEL WRITES to TimescaleDB (victory_market_db)**   ← must move, needs WRITE pool
```

### 3.3 Monolith components that *stay* but depend on live-data outputs

| Monolith component | Depends on | After extraction | Action needed |
|---|---|---|---|
| `controllers/price.controller.js` → `price.service.js` | Redis `PRICE:LATEST:*` | ✅ Keeps working (shared Redis, same keys) | None — document the contract |
| `controllers/indices.controller.js` → `indexCalculation.service.js` | Redis `PRICE:LATEST:*` / `Order_Depth:*` | ✅ Works *if* those keys are written | Verify depth keys are written at all (Defect/Error D) |
| `controllers/intervals.controller.js`, `history.controller.js`, `movers.controller.js` (+ their services) | `OHLC_*` candles in TimescaleDB | ✅ Works **only because** the aggregator moves with a WRITE pool | Defect 3 fix |
| `services/angel.service.js` (historical candles) | Angel session token via `angelAuth.service` | ❌ Crashes on boot (missing require) | Token-reader refactor (§7.4) |
| `controllers/broker.controller.js` | In-process `getBrokerManager()` | ❌ Reports "no brokers" (silent lie) | Proxy to live-data `/health` (§7.5) |
| `tradeRecommendations.controller.js`, `planSync.service.js`, `tradeExpireCron.js` | In-process notification functions | ❌ Silent notification loss | `NotificationClient` (§6.2) |

### 3.4 Clients (the contract owners)

| Client | Uses | Can we redeploy it instantly? |
|---|---|---|
| Admin panel (React, `D:\Admin-panel`) | REST `/api/*` on :3002, WS `/ws/prices`, Strategy Engine :3003 | Yes (web) |
| **Mobile app (VictoryApp)** | FCM pushes, user-feed endpoints (`getNotificationUserWise`, `mark-delivered`, `mark-opened`, `save-token`…), likely market WS | **No — installed on user phones** |
| Strategy Engine (:3003) | Redis `ohlc:candle_close` (from data pipeline), shared JWT secret | Yes, but no changes needed |

The mobile app is the reason the gateway (§5) is non-negotiable: its URLs are frozen.

---

<a name="4-target-architecture"></a>
## 4. Target architecture

```
                        ┌──────────────────────────────────────────────┐
   Admin Panel (web)    │              NGINX (one public origin)        │   Mobile App
   ─────────────────────►  path-based routing:                          ◄──────────────
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
                 │ billing, ~140    │  │ routes + crons │  │ BrokerManager,      │
                 │ admin routes,    │  │ FCM (firebase) │  │ MarketScheduler,    │
                 │ GraphQL, trade   │  │ stream consumer│  │ wsHub + both WS     │
                 │ recs, content    │  │ + DLQ          │  │ endpoints,          │
                 │                  │  │                │  │ realtimeAggregator  │
                 │ NotificationClient──┼──► Redis Stream │  │ (WRITE candles),   │
                 │ (XADD, drop-in)  │  │   notif:dispatch│ │ Angel auth OWNER    │
                 │ Angel token      │  └───────┬────────┘  └──┬──────────────────┘
                 │ READER           │          │              │
                 └────────┬─────────┘          │              │
                          │                    │              │
        ┌─────────────────▼────────────────────▼──────────────▼───────────┐
        │  PostgreSQL: victory_db + victory_market_db (shared, role-based)│
        │  Redis: PRICE:LATEST:* / Order_Depth:* / ohlc:candle_close      │
        │         / notif:dispatch (stream) / ANGEL:SESSION               │
        └──────────────────────────────────────────────────────────────────┘

   Strategy Engine :3003 — UNTOUCHED (already a separate service)
```

Communication rules:

| From → To | Pattern | Mechanism | Why |
|---|---|---|---|
| Clients → all services | Sync | HTTP/WS **through nginx only** | Frozen URLs, instant rerouting |
| Monolith → Notif svc (paths 3/4/5 triggers) | **Async fire-and-forget** | Redis Stream `notif:dispatch` | Durable: if notif svc is down, messages wait and are processed on recovery — trade flow never blocks on FCM |
| Admin panel → Notif svc (CRUD, send-now) | Sync | HTTP via nginx | Admin needs immediate response |
| Live-data → everyone | Fire-and-forget | Redis keys (`PRICE:LATEST:*`) + WS broadcast | Already the contract today |
| Monolith → Live-data (broker status) | Sync | Internal HTTP `/health` | Tiny, read-only |
| Anything → Angel One login | **Only live-data svc** | — | Single-writer rule (§7.4) |

---

<a name="5-gateway"></a>
## 5. The gateway layer

Deploy nginx **before any extraction**, routing 100% of traffic to the monolith. This is a pure pass-through with no behavior change, and it is the single highest-leverage step in the whole migration:

- **Zero client changes, forever.** Web and mobile keep one origin.
- **Per-route canary.** Move `/api/notification-query` alone to the new service while everything else stays — impossible with "edit the frontend URL".
- **Instant rollback.** Cutover and rollback are an upstream flip + `nginx -s reload` (sub-second, no deploys, no client involvement).

Reference config (the two WS-specific settings are the ones people forget):

```nginx
upstream monolith      { server 127.0.0.1:3002; }
upstream notif_svc     { server 127.0.0.1:3005; }
upstream livedata_svc  { server 127.0.0.1:3004; }

map $http_upgrade $connection_upgrade { default upgrade; '' close; }

server {
    listen 80;  # add TLS in production

    # WebSockets → live-data service (after Phase 3; before that, → monolith)
    location ~ ^/ws/(prices|market)$ {
        proxy_pass http://livedata_svc;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # MUST exceed the 30s server ping interval, or nginx kills idle sockets:
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
        client_max_body_size 50m;   # monolith accepts 50mb JSON bodies today
    }
}
```

Notes:
- Cookies pass through untouched (same origin → same cookie domain). JWT-in-cookie auth keeps working everywhere with zero changes.
- If you later add a second live-data fan-out instance, native `ws` needs **no sticky sessions** (no socket.io handshake state); clients re-`SUBSCRIBE` on reconnect, which `marketWs.js` already does.

---

<a name="6-notification-service"></a>
## 6. Notification Service — corrected design

### 6.1 What moves (Gemini's list, which was correct here)

`push.service.js`, `notificationBroadcast.service.js`, `autoNotification.service.js`, `autoNotificationCron.js`, the three notification controllers, the three notification route files, `config/firebase.js` + the service-account JSON, plus copies of `authenticateUser.js`, `authorizeAccess.js`, cookie-parser and the exact CORS config.

### 6.2 What the monolith KEEPS: the `NotificationClient` (the fix for Defects 1 & 2)

A single new file in the monolith, exporting **the same function names with the same signatures** as the code being removed:

```js
// src/services/notificationClient.js  (stays in the monolith)
const { getRedis } = require("../db/redisClient");
const { randomUUID } = require("crypto");

async function dispatch(action, payload, correlationId) {
  await getRedis().xAdd("notif:dispatch", "*", {
    action,                                  // e.g. "BROADCAST_BY_QUERY" | "PLAN_SYNC" | "USER_PLAN_SYNC"
    payload: JSON.stringify(payload),
    idempotency_key: payload.idempotencyKey || randomUUID(),
    correlation_id: correlationId || randomUUID(),
    ts: Date.now().toString(),
  });
}

// Drop-in replacements — same names/signatures as the old in-process functions:
exports.broadcastNotificationByQuery = (args) => dispatch("BROADCAST_BY_QUERY", args);
exports.notifyPlanChange            = (args) => dispatch("PLAN_SYNC", args);
exports.notifyUserPlanChange        = (args) => dispatch("USER_PLAN_SYNC", args);
```

Then exactly **four import lines change** in the monolith, and *no business logic changes at all*:

| File | Old import | New import |
|---|---|---|
| `controllers/tradeRecommendations.controller.js:6-7` | `require("../services/notificationBroadcast.service.js")` | `require("../services/notificationClient.js")` |
| `controllers/plansNew.controller.js:74` | `require("../services/planSync.service")` | `require("../services/notificationClient.js")` |
| `controllers/proPlanTransactions.controller.js:90` | same | same |
| `controllers/proPlanTransactions.controller.js:155` | same | same |

`tradeExpireCron.js` needs no change — it goes through the controller.

One behavioral nuance to accept and document: these triggers become **asynchronous** (enqueued, then sent by the service typically within milliseconds). For pushes to phones this is invisible — FCM delivery itself already takes seconds. None of the call sites read the send result for business decisions (they only `console.error` on failure — verify this during Phase 1 by checking each call site's use of the return value).

### 6.3 The service side: stream consumer

```js
// Consumer loop (notification-service/src/consumers/dispatchConsumer.js)
// XGROUP CREATE notif:dispatch notif_workers $ MKSTREAM   (once, at startup)
while (running) {
  const msgs = await redis.xReadGroup("notif_workers", consumerName,
      { key: "notif:dispatch", id: ">" }, { COUNT: 10, BLOCK: 5000 });
  for (const m of msgs) {
    try {
      await handle(m);                      // routes on m.action → existing service functions
      await redis.xAck("notif:dispatch", "notif_workers", m.id);
    } catch (err) {
      // retry counter in the message / PEL delivery count; after 5 failures:
      await redis.xAdd("notif:dispatch:dead", "*", { ...m, error: String(err) });
      await redis.xAck("notif:dispatch", "notif_workers", m.id);
      log.error({ correlationId: m.correlation_id }, "moved to DLQ");
    }
  }
}
```

- **At-least-once delivery** (crash before ACK ⇒ redelivery). That's why idempotency (next section) is mandatory, not optional.
- A startup sweeper claims pending messages from dead consumers (`XAUTOCLAIM` with `min-idle-time` 60s).
- The **DLQ stream** (`notif:dispatch:dead`) is monitored and alertable; a small admin endpoint can re-drive it.

### 6.4 Real idempotency (the fix for Error C)

Two database-level guarantees that make duplicate crons, redeliveries, and crash-retries all harmless:

**1. Atomic claim before sending a scheduled notification:**

```sql
UPDATE notifications
SET    status = 'PROCESSING', claimed_by = $1, claimed_at = now()
WHERE  id = $2 AND status = 'PENDING'
RETURNING id;
-- 0 rows returned ⇒ someone else claimed it ⇒ skip. Two crons are now harmless.
```

A sweeper resets rows stuck in `PROCESSING` for >10 minutes back to `PENDING` (crash recovery), and the per-user constraint below prevents re-sends to users who already got it.

**2. Unique constraint as the duplicate firewall:**

```sql
-- Check for existing duplicates first; then:
ALTER TABLE push_notifications
  ADD CONSTRAINT uq_push_notif_user UNIQUE (notification_id, user_id);
-- Sender does INSERT ... ON CONFLICT DO NOTHING and only pushes to FCM
-- for rows actually inserted.
```

With these two in place, the feature flag stops being a safety mechanism and becomes what it should be: a convenience switch.

### 6.5 Cron ownership

The every-minute scheduled-notification cron moves to the notification service. During transition, both processes *may* run it — the DB claim makes that safe. Keep the new boxes/containers on **Asia/Kolkata** time or rely on the code's explicit IST conversion (verify which it is before cutover); a UTC server clock would shift every scheduled notification by 5.5 hours.

---

<a name="7-live-data-service"></a>
## 7. Live Data Service — corrected design

### 7.1 The complete move list (corrected)

**Moves to live-data service:**

| Component | Note |
|---|---|
| `sockets/` — all 8 files | wsHub, priceSocket, index, internalMarket.websocket + instance, angelOne instance, sharekhan ×2 |
| `core/` — all 6 files | BaseBrokerWS, BrokerManager, BrokerRegistry, TickGate, MessageNormalizer, MarketScheduler |
| `brokers/dhan/` — all 4 files | self-contained |
| `services/marketData/angelAuth.service.js`, `angelLoginRunner.js`, `angelTokenScheduler.js`, `angelWebSocket.service.js`, `angelWebSocket.service.v2.js` | live-data svc becomes the **only** process that logs in to Angel One |
| **`services/marketData/realtimeAggregator.service.js`** | **Gemini missed it. Moves with a WRITE pool** (Defect 3) |
| `services/marketData/tickGenerator.service.js` | verify its consumers first; if it synthesizes ticks into the same pipeline, it moves |
| `config/brokerConfig.js`, `script/syncAngelOneTokens.js`, the 8:45 AM token cron from `index.js` | token lifecycle moves as one unit |
| `db/redisClient.js`, `db/timescaleClient.js` (copies) | own connections |

**Stays in the monolith (read-side market features):**

| Component | Why it stays |
|---|---|
| `intervals.controller.js` + `intervalCandles.service.js` | reads candles from DB (verify: read-only) |
| `history.controller.js` + `historyAggregation.service.js` | reads/aggregates candles from DB |
| `movers.controller.js` + `movers.service.js` | reads DB/Redis |
| `price.controller.js` + `price.service.js` | reads Redis `PRICE:LATEST:*` |
| `indices.controller.js` + `indexCalculation.service.js` | reads Redis |
| `angel.service.js` | historical-candle REST — **needs the token-reader refactor (§7.4)** |
| `broker.controller.js` | **becomes a proxy (§7.5)** |
| chart/sparkline/symbols/trading routes | DB reads |

**Decide before moving (likely dead):** `services/smartapiStream.js` — required by nothing. Phase 0 determines whether order-depth is a live feature; if dead, it is deleted, not migrated.

### 7.2 Contracts to freeze (write these down before touching anything)

These five contracts are what "nothing breaks" *means*. Capture real samples of each (golden files) in Phase 0 and diff against them at every step:

1. **`/ws/prices` frames:** `{ "type": "price", "data": {...} }` and `{ "type": "depth", "data": {...} }` — broadcast to all clients, no subscription needed.
2. **`/ws/market` protocol:** client sends `{ type: "SUBSCRIBE", page, context, symbols[] }` / `{ type: "UNSUBSCRIBE_DELAYED", ... }` (30s grace); server sends `{ type: "PRICE", symbol, ...payload }` plus Redis snapshots immediately on subscribe; only symbols enabled in `InternalTokenList.websocket_enabled` are broadcast.
3. **Redis keys:** `PRICE:LATEST:{symbol}` — exact JSON shape, 60s TTL; `Order_Depth:{token}` 24h TTL (if alive).
4. **TimescaleDB writes:** 1-min candle rows exactly as `insert1MinCandle` produces them today (same table, same conflict handling per Migration 028).
5. **Auth:** both WS endpoints are unauthenticated today. Preserve that (market data is public); do not "improve" it mid-migration.

### 7.3 Internal scaling seam (cheap now, priceless later)

Inside the new service, in addition to the existing in-process `pushPrice`/`broadcastPrice` calls, `BrokerManager._onTick` also `PUBLISH`es accepted ticks to a Redis channel `ticks:validated` (behind a flag, off by default). Cost: one line and one Redis publish per tick. Benefit: when one fan-out process is no longer enough, you can run N stateless "edge" WS instances that subscribe to `ticks:validated` — without ever touching broker ingest code again. Broker connections themselves must always be a **single instance** (broker sessions are singletons), so this ingest/fan-out split is the only honest horizontal-scaling story for this system.

### 7.4 Angel One token: single-writer rule (the fix for Defect 4)

- **Owner:** live-data service runs `AngelAuthService.login()`, the token scheduler, and the 8:45 AM IST refresh cron. Nobody else ever logs in with the system credentials.
- **Publication:** after each login/refresh, the service writes the session/feed tokens where they already live today (the credentials table) **and** mirrors them to Redis `ANGEL:SESSION` (with TTL) for cheap reads.
- **Monolith change:** `angel.service.js:2` swaps `require('./marketData/angelAuth.service.js')` for a ~20-line `angelTokenReader.js` that reads the token from Redis/DB and never refreshes. If a monolith call gets a 401 from Angel One, it re-reads the token (the owner will have refreshed it) — it must **not** trigger a login itself.
- Per-user broker tokens (the `save-token` / place-order flows) are unrelated to this — they are user-supplied sessions and stay in the monolith untouched (verified: `angelOne.controller.js` does not use `AngelAuthService`).

### 7.5 `broker.controller.js` becomes a proxy

```js
// monolith: controllers/broker.controller.js (after Phase 3)
exports.getBrokerStatus = async (req, res) => {
  const r = await axios.get(`${env.LIVE_DATA_URL}/health`, { timeout: 2000 });
  res.json(r.data.brokers);
};
```

Two seconds of work in the live-data service to expose the same shape `getBrokerManager().getHealth()` returns today; the admin panel sees an identical response.

### 7.6 Backpressure guard (one line, prevents the classic WS memory blowup)

```js
// in every broadcast loop:
if (ws.bufferedAmount > 1_000_000) return;  // skip slow client; next tick supersedes this one anyway
```

Price ticks are last-value-wins by nature — skipping a tick to a stalled client is strictly better than buffering unbounded memory until the process dies.

---

<a name="8-data-ownership"></a>
## 8. Database, Redis, and "who owns what"

### 8.1 Corrected ownership table

| Store / resource | Single writer | Readers | Gemini's version | Correction |
|---|---|---|---|---|
| `OHLC_*` 1-min candles (victory_market_db) | **Live-data svc** (realtimeAggregator) | Monolith (charts/history/movers), Strategy Engine | "live-data is read-only" | **WRITE pool required** |
| `PRICE:LATEST:{symbol}` (Redis) | Live-data svc | Monolith REST, internal WS | ✅ same | — |
| `Order_Depth:{token}` (Redis) | smartapiStream *(if alive)* | indexCalculation | listed as live | verify; likely dead |
| Angel system session (credentials table + `ANGEL:SESSION`) | **Live-data svc only** | Monolith `angel.service` (reader) | not addressed | single-writer rule |
| `notifications`, `push_notifications`, `push_devices`, `notification_types`, `notification_query` | Notif svc | Notif svc (monolith only via stream) | ✅ same | + unique constraint, + claim column |
| `notif:dispatch` / `notif:dispatch:dead` (Redis Streams) | Monolith produces / Notif svc consumes | — | dispatch only | + DLQ |
| `InternalTokenList`, `control_tradingdays` | Monolith (admin UI writes) | Live-data svc polls (30s / 15s) | ✅ same | — |
| `users`, permissions, billing, content tables | Monolith | Notif svc reads `users` for targeting | ✅ same | targeting via read-only role (§8.4) |

### 8.2 Connection-pool budget

| Service | victory_db | victory_market_db | Rationale |
|---|---|---|---|
| Monolith | 20 | 15 | unchanged primary writer |
| Notification svc | 5 | — | low write volume |
| Live-data svc | — | **10 read/write** | parallel candle inserts (the aggregator's actual behavior), plus polling reads |
| Strategy Engine | — | 20 | unchanged |
| **Total** | 25 | 45 | comfortably under PG's default 100 |

### 8.3 Why the DB stays shared (agreeing with Gemini, explicitly)

Splitting databases during extraction would force dual-writes, data migration, and cross-service JOIN replacements — three large risk multipliers — for zero functional gain. Shared DB with **enforced single-writer-per-table rules** (above) gives you the operational benefit (clear ownership) without the migration risk. Revisit a physical split only after both services have been stable for a quarter.

### 8.4 Security fix at the boundary: the SQL-targeting queries

`notification_query` rows are **raw SQL executed by the server** — that's SQL-injection-by-design if anyone with admin access (or a compromised admin account) can edit them. While extracting, run those queries through a dedicated PG role:

```sql
CREATE ROLE notif_targeting LOGIN PASSWORD '...' ;
GRANT SELECT ON users, push_devices /* + whatever targeting needs */ TO notif_targeting;
ALTER ROLE notif_targeting SET statement_timeout = '5s';
ALTER ROLE notif_targeting SET default_transaction_read_only = on;
```

The notification service uses **two** pools: the normal one for its own tables, and this locked-down one *exclusively* for executing targeting queries. A malicious or broken query can now read user emails at worst — it can no longer `DROP TABLE` or lock the database. This costs ~an hour and removes the single scariest thing in this codebase.

### 8.5 Database safety rules (read this before touching anything)

**Rule zero: no query in this plan is ever run automatically.** No AI tool, no script, no migration step executes SQL against your database on its own. Every statement below is run **by a human on your team**, in the listed phase, **after taking a backup** (`pg_dump` of `victory_db`), in a quiet window.

The entire migration needs only **three** SQL statements — all of them *additive* (they only ADD things; they are physically incapable of losing data):

| # | Statement (defined in) | What it does | Can it lose data? |
|---|---|---|---|
| 1 | `ALTER TABLE notifications ADD COLUMN claimed_by …, claimed_at …` (§6.4) | adds two empty bookkeeping columns | No — adding columns never touches existing rows |
| 2 | `ALTER TABLE push_notifications ADD CONSTRAINT uq_push_notif_user UNIQUE (notification_id, user_id)` (§6.4) | adds a duplicate-blocker for the future | No — it only *refuses future duplicates*. If old duplicates already exist, it simply **fails to apply, harmlessly**, and changes nothing |
| 3 | `CREATE ROLE notif_targeting … GRANT SELECT …` (§8.4) | creates a new **read-only** database login | No — it creates something new and touches nothing existing |

**About old duplicates (statement 2):** if `push_notifications` already contains duplicate rows, **do not delete them directly**. The safe order, done by hand:

1. Take a backup.
2. Copy the duplicates aside first: `CREATE TABLE push_notifications_dupes_backup AS SELECT …` (a copy — original rows untouched).
3. Only then remove them from the live table, **inside a transaction your team inspects and commits manually**.

This plan deliberately does **not** provide a delete script — that decision and that keystroke must stay with your team.

**Runtime queries are a different thing (and nothing new):** the notification service's claim query (`UPDATE notifications SET status='PROCESSING' …`) is something the *application* runs while operating — exactly the way today's code already runs `UPDATE … SET status='SENT'`. It touches only the `notifications.status` bookkeeping column, never user data, and it is the same kind of query your system executes every minute right now.

---

<a name="9-auth"></a>
## 9. Authentication

- **JWT verification is replicated, never proxied.** Notification service gets copies of `authenticateUser.js` + `authorizeAccess.js` + `cookie-parser`, and verifies with the same `JWT_SECRET` read from environment (move the hardcoded `victory_secret_key_123` and the session secret `secretKey1234` into env files during Phase 4 — and rotate them, since they've been in git).
- **Sessions stay monolith-only.** `express-session` + FileStore (`./sessions` on local disk) never leaves the monolith. Before cutover, verify no notification controller touches `req.session` (they appear to use JWT via `authenticateUser` — confirm with a grep for `req.session` in the three notification controllers).
- **CORS must match exactly.** The monolith runs `cors({ origin: true, credentials: true })`. The notification service must replicate this initially — same-but-different CORS is a classic source of "works in Postman, breaks in browser." Tighten to an explicit origin allowlist in Phase 4, all services at once.
- **WS endpoints stay unauthenticated** (their current contract; market prices are public data).
- **Internal endpoints** (live-data `/health` details, DLQ re-drive) bind to localhost/VPC or require a shared internal header token — they should not be reachable through nginx.

---

<a name="10-phases"></a>
## 10. The phased migration plan

Every phase ends with the system fully working, and every step's rollback is listed. **No step deletes code until the final phase.**

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

| # | Action | Rollback |
|---|---|---|
| 0.1 | **Deploy nginx routing 100% → monolith.** Point clients/DNS at it. Verify both WS endpoints work *through the proxy* (watch for the 30s-ping vs `proxy_read_timeout` interaction). | Point DNS back at :3002 |
| 0.2 | **Capture golden traffic:** log a full market day of `/ws/prices` and `/ws/market` frames to files; snapshot sample responses of every REST endpoint that will move; dump example `PRICE:LATEST:*` values and candle rows. These are your contract fixtures. | n/a |
| 0.3 | Add `/health` + `/ready` to the monolith. | remove |
| 0.4 | Write integration tests for **all five notification paths** (§3.1 — especially trade auto-exit and plan-sync, the ones Gemini didn't know existed), plus tick delivery, subscribe/unsubscribe, and candle insert. | n/a |
| 0.5 | **Settle the dead-code question:** is `smartapiStream` / order-depth live anywhere? Is `Order_Depth:*` ever written in prod Redis? Decide: delete or migrate. Also confirm which DB `control_tradingdays` lives in, and whether any notification controller uses `req.session`. | n/a |

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
│        ├─ flag OFF → calls the old functions directly (today's path)  │
│        │                                                              │
│        └─ flag ON  → writes a message into the Redis Stream ──────────┼──► Redis
│                                                                       │     │
│   in-process consumer reads the stream back ◄─────────────────────────┼─────┘
│        │                                                              │
│        ▼                                                              │
│   the SAME old send functions → Firebase → user's phone               │
└───────────────────────────────────────────────────────────────────────┘

WHY: we prove the "message pipe" works in real production for several days,
while everything still lives safely inside one process. If anything is off,
flip the flag back (seconds) and you are on today's exact path again.
```

| # | Action | Rollback |
|---|---|---|
| 1.1 | Add `notificationClient.js`; switch the 4 import sites (§6.2) to it. Behind flag `NOTIF_VIA_STREAM=false` it calls the old functions directly — **zero behavior change on day one**. | flip flag / revert imports |
| 1.2 | **Run the stream loop inside the monolith:** with `NOTIF_VIA_STREAM=true`, the monolith XADDs to `notif:dispatch` *and* consumes its own stream in-process, calling the same old functions. This proves the message schema, claim logic, retries, and DLQ **with zero new processes and zero network hops**. Soak for several days. | flip flag |
| 1.3 | DB groundwork: dedupe-check then add `uq_push_notif_user`; add `claimed_by/claimed_at` columns; deploy the claim-based cron logic (still in the monolith). | drop constraint/columns |
| 1.4 | Diff all golden-contract fixtures — must be byte-identical. | — |

> Why 1.2 matters: the Gemini plan's first network split was also its first test of the async path. Here, by the time a second process exists, the async machinery has already run in production for days.

### Phase 2 — Notification service extraction (weeks 3–4).

```
STEP 2.2 — move routes ONE SMALL GROUP at a time (nginx canary)

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

Exactly ONE consumer is active at a time (controlled by two flags).
And even if both accidentally ran together, the database claim (§6.4)
makes a double-send IMPOSSIBLE — that is lock-level safety, not flag-level hope.
```

| # | Action | Rollback |
|---|---|---|
| 2.1 | Stand up `notification-service` on :3005 (moved code, own pools, copied auth/CORS, Firebase JSON, IST timezone). Stream consumer **off**, cron **off**, REST **on**. | stop process |
| 2.2 | **Canary by route group via nginx**, lowest-risk first: (a) `/api/notification-query` → :3005, watch a day; (b) `/api/notifications` admin CRUD; (c) user-feed endpoints (**mobile-facing — watch 4xx/5xx rates closely here**); (d) `/api/notification/send*`. | flip the nginx line back (sub-second) |
| 2.3 | **Flip the consumer:** monolith in-process consumer off, service consumer on (flags). Trade auto-exit, expiry, and plan-sync pushes now flow monolith → stream → service → FCM. Run the Phase-0.4 integration tests for paths 3/4/5 specifically. | flip both flags back |
| 2.4 | **Flip the cron** to the service. Double-running during the overlap is harmless because of the DB claim. | flip back |
| 2.5 | **Bake 2 weeks.** Monolith's notification code is dormant behind flags but deployable in minutes. Then — and only then — delete it and the monolith's Firebase JSON. | (after deletion) git revert |

### Phase 3 — Live data extraction (weeks 5–7). The hard one — but without dual-run.

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
STEP 3.4 — the cutover, on a weekend evening
            (market CLOSED  =  zero live ticks anywhere  =  nothing can be missed)

   ①  monolith:   BROKERS_ENABLED=false  → restart
        └── monolith stops talking to Angel One / Dhan completely
   ②  live-data:  BROKERS_ENABLED=true
        └── live-data service logs in to Angel One —
            it is now the ONLY process anywhere that logs in
   ③  nginx:      /ws/prices + /ws/market  → :3004     (one reload, sub-second)
   ④  monolith:   LIVE_DATA_EXTRACTED=true
        └── the Angel token-reader and broker-status proxy switch on

   ROLLBACK = do ④ ③ ② ① in reverse. Under 5 minutes,
   still inside the closed window — users never see a thing.
   (And because nothing ever dual-runs, rollback has no session conflicts either.)
```

| # | Action | Rollback |
|---|---|---|
| 3.1 | Stand up `live-data-service` on :3004 with the **corrected move list** (§7.1) and **brokers disabled by flag**. It serves both WS endpoints and answers `SUBSCRIBE` with Redis snapshots — which still come from the monolith's ticks via shared Redis. This validates the entire WS/subscription layer against **live production data** while the monolith still owns ingest. Test via a parallel nginx path (`/ws-canary/prices`) or a test client pointed at :3004 directly. | stop process |
| 3.2 | **Replay validation in staging:** pipe the Phase-0.2 golden tick recordings through the service's full pipeline (TickGate → aggregator → broadcast). Diff produced candle rows and WS frames against fixtures. This replaces Gemini's impossible "dual-run parity comparison" with a deterministic one. | n/a |
| 3.3 | Ship the two monolith-side changes **dark** (inactive until cutover): `angel.service.js` → token-reader; `broker.controller.js` → proxy (behind `LIVE_DATA_EXTRACTED` flag). | flag off |
| 3.4 | **Cutover — in a market-closed window (weekend evening):** ① flag brokers OFF in monolith, restart; ② flag brokers ON in live-data service; ③ nginx flips `/ws/prices` + `/ws/market` → :3004; ④ enable `LIVE_DATA_EXTRACTED` in monolith. Angel login/8:45-cron/token-scheduler now run **only** in the service. Markets are closed: zero ticks are flowing, so nothing can be missed. | reverse ①–④ (minutes, still inside the closed window) |
| 3.5 | **War-room the next market open** (§14.2 checklist): broker connects at scheduler signal, ticks/s, `PRICE:LATEST` freshness, candle inserts appearing, admin-panel prices moving, monolith chart/movers/history endpoints fresh, frontend WS reconnects clean. | flags + nginx back; brokers restart in the monolith in <5 min. Safe **because we never dual-ran** — there is no session conflict on rollback either. |
| 3.6 | **Bake 2–4 weeks** with monolith socket/broker code dormant. Then delete `sockets/`, `core/`, `brokers/`, moved `marketData/` files from the monolith. | git revert |

### Phase 4 — Hardening (week 8).

Secrets to env/secret manager (+ rotate both hardcoded secrets), `notif_targeting` read-only role live, CORS allowlist everywhere, remove flags, dashboards + alerts (§12), delete confirmed-dead `smartapiStream`, update docs, and remove the now-unneeded `uncaughtException → process.exit` coupling note from the ops runbook (a monolith crash no longer kills market data — celebrate that in the postmortem culture).

---

<a name="11-risks"></a>
## 11. Risk register

| # | Risk | Phase | Likelihood | Impact | Mitigation |
|---|---|---|---|---|---|
| 1 | Silent loss of trade auto-exit / plan-sync pushes (Defects 1–2) | 2 | **Certain under Gemini plan** | High | `NotificationClient` + integration tests for paths 3/4/5 + FCM failure-rate alert |
| 2 | Candle writes stop (Defect 3) | 3 | **Certain under Gemini plan** | High | aggregator moves with WRITE pool; candle-insert-lag alert |
| 3 | Monolith boot crash on missing `angelAuth` require (Defect 4) | 3 | **Certain under Gemini plan** | High, loud | token-reader shipped dark before cutover |
| 4 | Angel session invalidation loops | 3 | High if dual-run | High | **no dual-run, ever**; single-writer token rule |
| 5 | Mobile app breaks on moved endpoints | 2 | High without gateway | High | nginx keeps every public URL identical |
| 6 | Duplicate notifications (double cron / stream redelivery) | 2 | Medium | Medium | DB claim + unique constraint (duplicates impossible by construction) |
| 7 | nginx kills idle WS connections | 0 | Medium | Medium | `proxy_read_timeout` > ping interval; verified in 0.1 |
| 8 | Notification service down → triggers lost | 2+ | Low | Medium | Stream buffers durably; PEL/backlog alert; messages process on recovery |
| 9 | Reconnect storm at WS cutover | 3 | Medium | Low | cutover in closed window; clients already auto-reconnect; add jitter to frontend backoff opportunistically |
| 10 | IST timezone wrong on new service host | 2 | Medium | Medium | explicit TZ check in deploy script + a scheduled-notification smoke test |
| 11 | `module-alias` paths break when files move | 2–3 | Medium | Low, loud | boot smoke test in CI for each service |
| 12 | Pool exhaustion from new services | 2–3 | Low | Medium | budget in §8.2; pool metrics + alerts |
| 13 | Order-depth feature was already dead and migration "breaks" it cosmetically | 3 | Medium | Low | Phase 0.5 settles its status before anyone blames the migration |

---

<a name="12-observability"></a>
## 12. Observability

**Correlation IDs:** nginx stamps `X-Correlation-Id: $request_id` on every request; services log it; the monolith puts it into every stream message (`correlation_id` field, §6.2); the notification service logs it on consume and on FCM send. One grep then traces *admin click → stream → FCM message ID*.

**Metrics that actually catch the failure modes in this document:**

| Metric | Service | Alert | Catches |
|---|---|---|---|
| `tick_e2e_lag_ms` p50/p99 (broker `exchange_ts` → `ws.send`) | live-data | p99 > 2s for 1 min (market hours) | slow pipeline, GC stalls |
| ticks/sec per broker | live-data | < 1 for 30s during market hours | dead broker connection |
| broker state | live-data | `disconnected` > 60s in market hours | session/auth issues (risk 4) |
| **candle insert rate + lag** | live-data | 0 inserts for 3 min in market hours | **Defect-3 class failures** |
| TickGate reject % | live-data | > 50% for 5 min | upstream data corruption |
| WS clients + `bufferedAmount` drops | live-data | clients = 0 for 2 min in market hours | proxy/cutover mistakes |
| `notif:dispatch` PEL size + consumer lag | notif | PEL > 100 or lag > 60s | stuck/dead consumer |
| DLQ depth | notif | > 0 | poisoned messages |
| FCM failure rate | notif | > 10% over 5 min | credential/token issues, **silent-loss class failures** |
| cron last-claim timestamp | notif | > 3 min stale | dead scheduler |
| pool idle/waiting per service | all | 0 idle for 30s | risk 12 |
| Redis `PRICE:LATEST:*` freshness (sampled) | monolith side | age > 90s in market hours | end-to-end canary from the *consumer's* viewpoint |

**Logging:** structured JSON in prod, one format across services. Live-data logs state changes and rejections only (never per-tick); notification service logs every send attempt with correlation ID and FCM message ID.

---

<a name="13-testing"></a>
## 13. Testing strategy

1. **Golden contract tests** (built in Phase 0.2, run at every phase): recorded WS frames, REST response shapes, Redis value shapes, candle rows — diffed byte-for-byte where deterministic, schema-checked where not.
2. **The five-path notification matrix** — each of §3.1's paths gets an end-to-end test: trigger → (stream) → `push_notifications` row → FCM call (mocked in CI, one real device in staging). Paths 3/4/5 are the regression tests Gemini's plan had no way to write, because it didn't know they existed.
3. **Idempotency tests:** double-deliver a stream message → exactly one push; run two crons concurrently → exactly one claim; kill the consumer mid-batch → redelivery → still one push.
4. **Replay tests** for live data (Phase 3.2): golden ticks in → identical candles and frames out.
5. **Chaos drills in staging:** kill the notification service for 10 minutes during sends (messages wait in stream, drain on restart); kill live-data (clients reconnect, monolith REST unaffected, brokers reconnect per BaseBrokerWS state machine); restart Redis (both services reconnect; verify stream group survives — it does, streams are persistent if Redis has persistence enabled — **check `appendonly`/RDB config**, since dispatch durability now depends on it).
6. **A full staging dress rehearsal of the Phase 3.4 cutover**, timed, before doing it in production.

---

<a name="14-runbooks"></a>
## 14. Runbooks

### 14.1 Live-data cutover (market-closed window)

```
[ ] Confirm market is closed and next open is > 12h away
[ ] Staging dress rehearsal passed within the last week
[ ] Golden fixtures current; replay test green
[ ] Monolith: BROKERS_ENABLED=false → restart → confirm no broker logins in logs
[ ] Live-data: BROKERS_ENABLED=true → confirm Angel login succeeds ONCE, token mirrored to ANGEL:SESSION
[ ] nginx: flip /ws/prices + /ws/market upstreams → reload → test client connects, snapshot replies arrive
[ ] Monolith: LIVE_DATA_EXTRACTED=true (token-reader + broker proxy active)
[ ] Verify: admin panel loads, charts/history/movers serve (stale-but-served is correct while closed)
[ ] Schedule war-room for next market open
```

### 14.2 First-market-open war room

```
[ ] MarketScheduler flips to OPEN; brokers auto-connect (BaseBrokerWS state log)
[ ] ticks/s > 0 within 60s of open;  tick_e2e_lag p99 < 2s
[ ] PRICE:LATEST freshness < 90s (checked FROM the monolith — consumer's viewpoint)
[ ] Candle inserts appearing in TimescaleDB (the Defect-3 check)
[ ] Admin panel: live prices moving; charts updating; movers fresh
[ ] broker.controller proxy returns real broker health
[ ] No Angel auth errors in EITHER service's logs (single-writer check)
[ ] WS client count ≈ pre-migration baseline
```

### 14.3 Rollback (any phase)

| Phase | Action | Time |
|---|---|---|
| 0–1 | flip flag (`NOTIF_VIA_STREAM=false`) | seconds |
| 2 (routes) | flip nginx upstream line, reload | seconds |
| 2 (consumer/cron) | flip two flags; DB claim makes the overlap safe | < 1 min |
| 3 | reverse the four cutover steps; brokers restart in monolith | < 5 min, no session conflicts because nothing dual-runs |
| post-deletion | git revert of the deletion commit + redeploy | < 30 min — which is why deletion waits for the bake period |

---

<a name="15-structure"></a>
## 15. Folder structure and deployment

Gemini's monorepo layout (its §12.1) is broadly fine. Corrections only:

- `live-data-service/src/services/marketData/` **additionally contains** `realtimeAggregator.service.js` (+ `tickGenerator.service.js` if Phase 0.5 confirms it's in the tick path), and `db/` has a **read/write** `timescaleClient.js`.
- `live-data-service` does **not** contain `smartapiStream.js`/`price.service.js` unless Phase 0.5 proves depth is live (`price.service` is a read-side monolith service — Gemini misplaced it).
- `main-backend` **gains** two small files Gemini's structure lacks: `services/notificationClient.js` and `services/marketData/angelTokenReader.js`.
- `notification-service` gains `consumers/dispatchConsumer.js`, `consumers/dlqRedrive.js`, and a second locked-down pool `db/targetingDb.js`.
- Add `gateway/nginx.conf` to the repo root — the routing table *is* the migration state; it must be version-controlled.

**Deployment reality check:** today this stack runs as bare Node processes (nodemon) on a Windows/EC2 box — not Docker. The docker-compose target is right for the future, but don't couple the migration to a containerization project (two migrations at once is how both fail). A PM2 ecosystem file gives process management, restart-on-crash, and log capture *now*:

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

Containerize in a separate, later project once the service boundaries have proven stable.

---

<a name="16-bites"></a>
## 16. Things that will bite you (corrected and extended)

Gemini's §14.7 list was its best section — items 1–10 there are real (cookie-parser, redis-vs-ioredis, CORS, Firebase JSON path, MarketScheduler/BrokerManager singletons, InternalTokenList pool, broadcastPrice chain, graceful shutdown, IST cron). Keep all of them, **plus** the ones it missed:

11. **`angel.service.js:2`** requires a file you're moving — the monolith won't boot (Defect 4). Ship the token-reader *before* cutover day.
12. **The four notification import sites** (§6.2 table) — miss one and that flow silently dies. Grep for `notificationBroadcast.service`, `planSync.service`, and `config/firebase` in the monolith after the swap; the only remaining hit should be `notificationClient.js` itself.
13. **`push_notifications` may already contain duplicates** — the unique constraint will fail to apply until you clean them. Check before Phase 1.3, not on the night you need it. If duplicates exist, follow the archive-first procedure in §8.5 — never a bare `DELETE` on production rows.
14. **nginx `proxy_read_timeout` vs the 30s WS ping** — default 60s is uncomfortably close; set 120s+ or you'll chase phantom disconnects.
15. **`module-alias` in the monolith's package.json** — moved files that relied on alias paths resolve differently in the new repos. A boot smoke test per service in CI catches this in seconds.
16. **Redis persistence** — notification dispatch durability now depends on Redis surviving restarts. Confirm AOF or RDB is enabled; default Windows Redis 5 configs often have weak persistence.
17. **`client_max_body_size` in nginx** — the monolith accepts 50 MB JSON; nginx defaults to 1 MB and will 413 the market-data imports the day after you add the proxy. Set it in Phase 0.1.
18. **The `uploads/` static directory and `charting_library-master`** are served by the monolith from local disk — they stay with it; don't route those paths anywhere else.
19. **`broker.controller.js`** returns a confidently empty answer after extraction instead of erroring (Defect 5) — flip it to the proxy in the same deploy as cutover, not "later".
20. **Don't "fix" things mid-move.** Every refactor you bundle into the migration multiplies the diff you must reason about when something breaks at 9:15 AM on the first market open. Move ugly code as-is; schedule beautification for after the bake period.

---

## Final implementation order (one page)

```
Week 1   Phase 0: nginx pass-through · golden traffic capture · health checks
         · 5-path integration tests · dead-code verdicts (smartapiStream, req.session, control_tradingdays)
Week 2   Phase 1: NotificationClient + 4 import swaps · in-monolith stream loop (flagged)
         · DB claim columns + unique constraint
Weeks 3–4 Phase 2: notif-svc up · nginx canary per route group (query → CRUD → feed → send)
         · consumer flip · cron flip · 2-week bake → delete monolith notif code
Weeks 5–7 Phase 3: live-data-svc up (brokers off, WS validated on live Redis) · golden replay
         · dark-ship token-reader + broker proxy · market-closed cutover · war-room first open
         · 2–4-week bake → delete monolith socket/broker code
Week 8   Phase 4: secrets rotated · notif_targeting role · CORS allowlist · alerts live · docs
```

**What you must NOT touch** (unchanged from Gemini, it was right): the Strategy Engine and its Signal WS, the data pipeline / `ohlc:candle_close` flow, JWT issuance, database schemas (beyond the two additive notification changes), and any business logic beyond the explicitly listed import swaps and the two monolith-side adapters.
