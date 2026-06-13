# 02 — The Full Architecture (Frontend ↔ Backend ↔ Database)

This file is the "big picture". After reading it you should be able to point
to **any feature** and explain, "the click goes here, then here, then here,
the data ends up in this table, and the result comes back through these
layers". The next file (03) zooms into the code; this file zooms out.

---

## 1. The complete system at a glance

```
┌────────────────────────────────────────────────────────────────────────────┐
│ BROWSER  (the admin user)                                                  │
│ ┌────────────────────────────────────────────────────────────────────────┐ │
│ │ Admin Panel (React, http://localhost:3000)                             │ │
│ │   • pages: Strategies / StrategyDetail / BacktestPage / Signals        │ │
│ │   • hooks: useStrategies, useSignals, useRealtimePrices, useApi        │ │
│ │   • services: strategyEngineService → strategyEngineApi (axios)        │ │
│ │   • singleton WebSocket: marketWs (prices) + per-page WS (signals)     │ │
│ │   • Redux slice: permissions                                           │ │
│ └─────┬──────────────────────────────────────────────────┬───────────────┘ │
└───────┼──────────────────────────────────────────────────┼─────────────────┘
        │  REST + cookie (JWT)                             │  REST + cookie
        │  WebSocket (signals)                             │  WebSocket (prices)
        ▼                                                  ▼
┌────────────────────────────────┐         ┌──────────────────────────────────┐
│ STRATEGY ENGINE  :3003          │         │ MAIN BACKEND  :3002              │
│ (this repo — Trade-Backend)     │         │ (separate repo, not on this PC)  │
│                                 │         │                                  │
│  Express + WS                   │         │  Login (issues JWT cookie)       │
│  Reads JWT cookie issued by 3002│         │  Users, KYC, billing,            │
│  Shares JWT_SECRET with 3002    │         │  market-data import,             │
│                                 │         │  /ws/prices (live LTP feed)      │
│                                 │         │  /me/permissions                 │
└──────┬─────────────────┬────────┘         └──────────────┬───────────────────┘
       │                 │                                 │
       │ SQL             │ Redis SUBSCRIBE                 │ SQL
       ▼                 ▼                                 ▼
┌─────────────────────────────────────────────┐   ┌──────────────────────────┐
│ PostgreSQL — victory_market_db              │   │ PostgreSQL — victory_db  │
│                                             │   │  users, permissions,     │
│ Owned by Strategy Engine:                   │   │  invoices, news...       │
│   • strategies          (one row/strategy)  │   └──────────────────────────┘
│   • strategy_symbols    (per-symbol state)  │
│   • signals             (every signal fired)│
│                                             │
│ Owned by co-developer's pipeline (READ-only │
│ for historical data; we UPSERT live):       │
│   • "OHLC_1MIN" .. "OHLC_1HOUR"  (candles)  │
│   • feature_table_1min .. _1hour (indicator │
│                                  values)    │
└────────────────────▲────────────────────────┘
                     │ writes candles, publishes
                     │ "ohlc:candle_close" to Redis
                     │
            ┌────────┴────────┐
            │ Redis (pub/sub) │
            │  channel:        │
            │  ohlc:candle_close│
            └────────▲────────┘
                     │
            ┌────────┴─────────────────────┐
            │ Co-developer's data pipeline │
            │ (separate service — NOT this │
            │  repo). Pulls quotes from the│
            │  broker, builds OHLC candles,│
            │  publishes close events.     │
            └──────────────────────────────┘
```

### Who owns what

| Component | Code lives in | Started by | Talks to |
|-----------|---------------|------------|----------|
| Admin Panel (React) | `D:\Admin-panel\Victory_Admin_Pannel` | `npm run dev` (port 3000) | Both backends + WS |
| Strategy Engine | `D:\Trade-Backend\Backend` | `npm start` (port 3003) | victory_market_db, Redis, browser WS |
| Main backend | separate repo | (already running on 3002) | victory_db, market WS |
| Data pipeline | separate repo | runs continuously | Broker quotes → victory_market_db → Redis |
| PostgreSQL | external server | always on | All backends |
| Redis | external server | always on | Pipeline (pub), Engine (sub) |

The Strategy Engine **never calls the Main backend**, and the Main backend
never calls the Strategy Engine. They cooperate purely by:
1. sharing the JWT secret (so a cookie issued by 3002 is honored by 3003),
2. sharing the `victory_market_db` database.

---

## 2. The two backends — why are they separate?

The admin panel screens like Users / KYC / Billing / News live on the Main
backend (:3002). The Strategy Engine (:3003) only handles **strategies,
signals, indicators, backtests**. Why split?

1. **Independent deploys.** A signal-engine deploy doesn't risk breaking
   login.
2. **Independent crashes.** A bug in the candle handler can never take down
   user management.
3. **Load isolation.** Candle processing can spike under volatile markets;
   keeping it in its own process protects API latency for everything else.
4. **Cleaner code.** Trading logic is complex; isolating it in one repo with
   its own conventions (`constants.js` as single source of truth, models =
   SQL, services = brain) keeps it understandable.

The cost is two URLs in the frontend — handled by two axios instances
(`api` for 3002, `strategyApi` for 3003) with the same cookie-auth setup.

---

## 3. The two databases — `victory_db` vs `victory_market_db`

| Database | Used by | Tables that matter here |
|----------|---------|-------------------------|
| `victory_db` (the "app DB") | Main backend | users, permissions, invoices, news, … |
| `victory_market_db` (the "market DB") | **Strategy Engine + data pipeline** | `strategies`, `strategy_symbols`, `signals` (ours) — `OHLC_*` and `feature_table_*` (pipeline's) |

The Strategy Engine's `db/appDb.js` connects to `victory_db` but is currently
unused — all models call `getMarketPool()` (→ `victory_market_db`). Why?
Because the engine needs to join its own tables with the candle/indicator
tables in one query. Keeping everything in one DB makes that possible.

The data pipeline created `OHLC_*` and `feature_table_*` with **capitalised
SQL identifiers** (`"OHLC_5MIN"`, `"Close"`, `"Symbol"`, `"Time"`). That
casing is forever quoted in our SQL.

---

## 4. The big data-flow stories (one diagram per scenario)

### Story A — User creates a strategy (button click → row in DB)

```
[Browser] Strategies page  →  click "+ New Strategy" → fill StrategyForm → Save
   │
   │ axios POST  /api/strategies  (cookie attached)
   ▼
[Strategy Engine :3003]
   app.js                    ──  cors, json, cookieParser, request log
   /api router               ──  matches /strategies
   strategies.routes.js      ──  auth middleware (JWT verify)
                             ──  validate(createStrategySchema)   (Joi)
                             ──  ctrl.create
   strategies.controller.js  ──  builds payload, attaches req.user.id
   strategy.model.js         ──  BEGIN
                                 INSERT INTO strategies ... RETURNING *
                                 INSERT INTO strategy_symbols (one per symbol)
                                 COMMIT
   utils/response.js         ──  res.status(201).json({ success:true, data })
   │
   │ 201 Created  { success, data: { id, name, symbols: [...] } }
   ▼
[Browser]
   useStrategies.createStrategy → toast.success → re-fetch list
```

### Story B — A candle closes → a live signal arrives in the browser

```
[Data pipeline]    builds the 10:30 5-minute candle, INSERTs into "OHLC_5MIN",
                   publishes Redis  ohlc:candle_close  { symbol, timeframe, ts }
                                                    │
                                                    ▼
[Strategy Engine] jobs/candleCloseHandler.js receives the message
   IndicatorService.compute(symbol, '5m')
       indicator.model.getOhlcCandles  (read recent closes)
       talib.execute(SMA, 10/20)        (math)
       indicator.model.upsert           (write sma_10, sma_20 into feature_table_5min)
   ConditionEngine.evaluate(symbol)
       strategy.model.findActiveBySymbol(symbol)   (every active strategy on this symbol)
       buildLiveIndicatorMap(symbol, [timeframes]) (latest + previous indicator rows)
       FOR EACH strategy:
           if IDLE  → evaluateConditions(entry)
                     match? → transitionToTriggered (atomic CAS IDLE→TRIGGERED)
                              → SignalService.create({ signal_type: 'ENTRY', ... })
           else (TRIGGERED) → check TSL  → if hit, CAS reset & EXIT signal
                              else check Target → if hit, CAS reset & EXIT signal
                              else evaluateConditions(exit_conditions) → CAS reset & EXIT
       SignalService.create
           signal.model.create  (INSERT INTO signals ...)
           broadcastFn(user_id, { type:'NEW_SIGNAL', data: signal })
   ws/signalWs.js
       userConnections.get(user_id).forEach(ws.send)
   │
   │ WebSocket frame { type:'NEW_SIGNAL', data: {...signal row...} }
   ▼
[Browser]
   useSignals.connectWebSocket().ws.onmessage
       setSignals(prev => [signal.data, ...prev])   ← appears at the top instantly
```

This is the **core loop** of the whole engine. Read it three times — every
other story is a variation of it.

### Story C — User runs a backtest

```
[Browser] StrategyDetail → pick dates+symbol → "Run Backtest"
       useNavigate(`/strategies/:id/backtest`, { state: ... })
       BacktestPage auto-runs (state.autoRun)
       axios POST /api/strategies/:id/backtest  { start_date, end_date, symbol }
   ▼
[Strategy Engine]
   strategies.controller.backtest
   BacktestService.run(strategyId, userId, startDate, endDate, symbol)
       strategy.model.findById                (ownership check)
       validate range (≤30 days, valid dates)
       collectTimeframes() over conditions+exit_conditions
       for each timeframe:
           indicator.model.getHistoricalRange (JOIN feature_table * OHLC for closes)
       loop over the DRIVER timeframe's candles:
           for each non-driver timeframe, advance a POINTER while its ts ≤ driver ts
           build indicatorMap for THIS candle
           if not in trade:
               evaluateConditions(entry) → record an ENTRY row (with sizing + brokerage if configured)
           else:
               check TSL  → exit at the stop level (mirrors live priority)
               else check Target → exit at the target level
               else check exit_conditions → exit at current close
       computePnLSummary  (win rate, profit factor, etc.)
   res.json({ success, data: { matches, summary, indicatorColumns, pnl_summary, skipped_trades } })
   ▼
[Browser] BacktestPage renders matches table + P&L card
```

Note: backtest is **per-symbol** (the user picks one). The same strategy can
own up to 20 symbols, but the backtest analyzes one at a time so results are
clean.

### Story D — User toggles a strategy active/inactive

```
PATCH /api/strategies/:id/toggle
   strategy.model.toggleActive       (UPDATE is_active = NOT is_active)
   strategy.model.forceResetTriggerState   (all symbols → IDLE)
   res.json({ ...strategy with symbol_states all IDLE... })
```

No synthetic EXIT signals here (toggling is a benign on/off). Policy B
(synthetic EXITs) only fires when **rule logic** changes — see Story E.

### Story E — User edits rule logic on a live (TRIGGERED) strategy

```
PUT /api/strategies/:id  { conditions: [new], action: 'SELL', ... }
   ruleLogicChanged = action || conditions || exit_conditions
                      || trailing_stop_loss || target  changed
   if ruleLogicChanged:
       triggeredSymbols = findTriggeredSymbols(id)     (before mutation)
       oldStrategy      = findById(id, userId)         (keep old action)
   strategy.model.update           (apply edits in a transaction)
   for each previously triggered symbol:
       reset = resetTriggerState   (CAS TRIGGERED→IDLE)
       if !reset: continue          (live engine just exited it — fine)
       fetch last 1m candle close
       SignalService.create({ signal_type:'EXIT', action_type: flip(oldAction),
                              trigger_price: close,
                              condition_snapshot: { triggered_by:'STRATEGY_EDITED', ... }})
```

This is "Policy B" in the comments. It guarantees **no open position is ever
orphaned** by a rule edit.

### Story F — User opens Signals page; live signals stream in

```
[Browser]
   /signals route mounts
   useSignals.fetchSignals(page, limit, filters)   → GET /api/signals  (initial page)
   useSignals.connectWebSocket()                   → ws://...:3003/ws  (cookie auth)
       ws.onmessage  →  prepend new signal, total++
       ws.onclose    →  setTimeout(connect, 3000)  (auto-reconnect)
   on unmount → disconnectWebSocket (clears timer, nulls onclose, then close)
```

---

## 5. The "request triangle" inside the backend

Every REST call follows this shape:

```
┌─ HTTP request ──────────────────────────────────────────────────┐
│                                                                 │
│  app.js               cors + json + cookieParser + request log  │
│        │                                                        │
│        ▼                                                        │
│  routes/index.js      mount feature routers under /api          │
│        │                                                        │
│        ▼                                                        │
│  routes/X.routes.js   auth → validate(schema) → controller fn   │
│        │                                                        │
│        ▼                                                        │
│  controllers/X.ctrl   parse req → call models/services          │
│        │                                                        │
│        ▼                                                        │
│  models/  +  services/                                          │
│  models = SQL,        services = business logic                 │
│        │                                                        │
│        ▼                                                        │
│  utils/response.js    consistent envelope                       │
│        │                                                        │
│        ▼                                                        │
│  HTTP response                                                  │
└─────────────────────────────────────────────────────────────────┘
```

If anything throws or `next(err)`s, the chain skips to
`middleware/errorHandler.js`, which converts the error to JSON with the right
status code.

---

## 6. The frontend's "data triangle"

Every page that talks to the backend does so via the same pattern (encoded in
the panel's own CLAUDE.md):

```
page (pages/X.jsx)
   │  imports
   ▼
custom hook (hooks/useX.js)        ←  uses useApi() for loading/error
   │  imports
   ▼
service (services/xxxService.js)   ←  thin function objects
   │  imports
   ▼
axios instance (services/xxxApi.js)  ←  baseURL, withCredentials, interceptors
```

For the Strategy Engine specifically:
- axios instance: `strategyEngineApi.js` → `${REACT_APP_STRATEGY_ENGINE_URL}api`.
- service: `strategyEngineService.js` (the function map mirroring backend routes).
- hooks: `useStrategies`, `useSignals`.
- pages: `Strategies`, `StrategyDetail`, `BacktestPage`, `Signals`.

The same pattern but for the Main backend uses `api.js` (`services/api.js`)
and there are many feature services (`userService.js`, `grievanceService.js`,
…). The two **never mix**; e.g. `useStrategies` only ever calls
`strategyEngineService`.

The **WebSocket** layer is special: it lives in `src/ws/` (singletons), not
under `services/`, and the `marketWs` is connected once in `App.js` whereas
the **signal WS** is connected per-page (Signals page on mount, disconnected
on unmount). The signal WS uses the Strategy Engine URL; the market WS uses
the Main backend URL — completely independent connections.

---

## 7. Authentication flow (single sign-on by shared secret)

```
1. User opens admin panel, types email/password.
2. Frontend → POST main-backend:3002/api/login  (credentials).
3. Main backend verifies → signs a JWT with JWT_SECRET → sets it as a cookie
   (HttpOnly, Domain=...).
4. Browser stores the cookie. localStorage["userSession"] is set so the SPA
   knows "we're logged in".
5. From now on, every cross-origin request from the panel includes the cookie
   (because both axios instances use withCredentials: true and CORS allows it).
6. When the panel calls strategy-engine:3003, the auth middleware reads the
   cookie's JWT, jwt.verify(token, JWT_SECRET) succeeds because both
   backends share the same secret, and req.user gets populated.
7. The Signals page opens a WebSocket to strategy-engine:3003/ws — the same
   cookie is sent in the upgrade handshake; signalWs.extractToken finds it and
   verifies.
```

That's why `.env.example` says **"JWT_SECRET — same secret as main backend
:3002"**: if the two secrets differ, the panel will look logged in but every
call to the Strategy Engine returns 401.

---

## 8. The lifecycle of a strategy (state across all components)

```
              CREATE (POST /strategies)
                       │
                       ▼
            ┌──────────────────────┐
            │ strategies row       │
            │  is_active = false   │   ←──── default off until user toggles ON
            │ + N strategy_symbols │
            │   trigger_state=IDLE │
            └──────────────────────┘
                       │
        toggle ON  →  is_active = true
                       │
                       ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Engine includes this strategy in findActiveBySymbol         │
   │  every time a candle of any matching symbol closes.          │
   └──────────────────────────────────────────────────────────────┘
                       │
            entry conditions match
                       │
                       ▼
       strategy_symbols: IDLE → TRIGGERED + ENTRY signal
                       │
        ┌──────────────┼───────────────┐
        ▼              ▼               ▼
       TSL hit     Target hit    Exit conditions match
        │              │               │
        └──────────────┴───────────────┘
                       │
       strategy_symbols: TRIGGERED → IDLE + EXIT signal
                       │
                       ▼
            (next entry trigger can fire again)
```

Edits ("rule logic") fire Policy B (Story E). Toggle off → force reset all
symbols to IDLE (no synthetic exit). Delete → row gone (FK strategy on
strategy_symbols and signals: signals are kept by design — "history
preserved").

---

## 9. The lifecycle of a single signal

```
ConditionEngine decides to fire        SignalService.create
       │                                       │
       │                                       ▼
       │                              INSERT INTO signals
       │                              VALUES (..., status='pending')
       │                              RETURNING *
       │                                       │
       │                                       ▼
       │                              broadcastFn(user_id, NEW_SIGNAL)
       │                                       │
       │                                       ▼
       │                                  ws.send → browser
       ▼                                       │
   (status stays 'pending';                   ▼
    the trade-execution module                Browser: signal pops in feed
    that would flip it to 'executed'          (and on the strategy detail)
    is a future module — not in this repo)
```

The signal table also keeps `condition_snapshot` — the indicator values that
caused this fire. That's what powers the eventual "why did this fire?" panel
on the frontend and is invaluable for debugging.

---

## 10. End-to-end summary diagram (everything together)

```
                  ┌───────────────────────┐
                  │      ADMIN USER       │
                  └──────────┬────────────┘
                             │ HTTPS
                             ▼
         ┌────────────────────────────────────────┐
         │   ADMIN PANEL (React, port 3000)       │
         │                                        │
         │ pages → hooks → services → axios       │
         │      ╲                ╲                │
         │       ╲                ╲—— marketWs ──► main backend /ws/prices
         │        ╲—— signalWs ───────────────────► strategy engine /ws
         └────┬───────────────────────────────┬───┘
              │ REST + cookie                 │ REST + cookie
              ▼                               ▼
       ┌────────────┐                  ┌──────────────────┐
       │ MAIN :3002 │                  │ STRATEGY :3003   │
       │  Login     │                  │  /api/strategies │
       │  Users     │                  │  /api/signals    │
       │  /me/perms │                  │  /api/indicators │
       │  /ws/prices│                  │  /ws (signals)   │
       └─────┬──────┘                  └─────┬───┬───┬────┘
             │                                │   │   │
             │ SQL                            │   │   │ SUBSCRIBE
             ▼                                │   │   ▼
       ┌──────────┐                           │   │  ┌──────┐
       │victory_db│                           │   │  │REDIS │◄── PUBLISH "ohlc:candle_close"
       └──────────┘                           │   │  └───▲──┘
                                              │   │      │
                                       SQL    │   │      │
                                              ▼   ▼      │ INSERT candle / publish event
                                  ┌──────────────────────┴────┐
                                  │ victory_market_db          │
                                  │  • strategies              │
                                  │  • strategy_symbols        │
                                  │  • signals                 │
                                  │  • OHLC_* (candles)        │
                                  │  • feature_table_* (indic) │
                                  └────────────▲───────────────┘
                                               │
                                  ┌────────────┴───────────┐
                                  │  Data pipeline service │
                                  │  (pulls broker quotes, │
                                  │   builds candles,      │
                                  │   publishes closes)    │
                                  └────────────────────────┘
```

You now have the full mental model. Next file zooms into the actual code,
file by file, with the architecture above as the map.

**Next:** [03_BACKEND_CODE_WALKTHROUGH.md](03_BACKEND_CODE_WALKTHROUGH.md)
