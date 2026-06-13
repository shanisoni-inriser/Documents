# 03 — Backend Code Walkthrough (file by file, line by line where it matters)

Now we go through **every backend file** in a sensible order: entry point →
config → middleware → routes → controllers → models → services → utils → ws →
jobs → migrations. For each file:

1. **What this file is for** (one sentence).
2. **Walk through the code in execution order**.
3. **The "gotchas"** — the non-obvious bits the original author left comments
   about, with the *why* in plain language.

When line numbers are referenced, they match the code you have in the repo
at the moment this document was written. If you edit a file the numbers can
shift — read the surrounding context, not the number.

---

## 0. Boot order — what happens when you run `npm start`

```
node server.js
  ├─ require('./src/app')
  │     ├─ require('./routes')
  │     │     ├─ require('./strategies.routes') → require controllers, validators
  │     │     │     ├─ require('./controllers/strategies.controller')
  │     │     │     │     ├─ require('./models/strategy.model') → require('./db/marketDb')
  │     │     │     │     ├─ require('./models/signal.model')
  │     │     │     │     ├─ require('./models/indicator.model')
  │     │     │     │     ├─ require('./services/BacktestService')  (loads ConditionEngine etc.)
  │     │     │     │     └─ require('./services/SignalService')
  │     │     │     └─ require('./validators/strategy.validator')
  │     │     ├─ require('./signals.routes') → controllers/signals.controller
  │     │     └─ require('./config.routes') → controllers/config.controller
  │     ├─ require('./middleware/errorHandler')
  │     ├─ require('./config/env')   ← validates .env, exits if anything missing
  │     └─ require('./config/logger')
  ├─ require('./src/config/env')
  ├─ require('./src/config/logger')
  ├─ require('./src/ws/signalWs')
  ├─ require('./src/services/SignalService')
  └─ require('./src/jobs/candleCloseHandler')

server = http.createServer(app)
signalWs.init(server)                            ← WS attached
SignalService.setBroadcast(signalWs.broadcastSignal)  ← DI wiring
candleCloseHandler.start()                       ← Redis subscribe begins
server.listen(3003)                              ← API + WS both up
```

If any **required env var** is missing, `env.js` calls `process.exit(1)`
during the very first `require`. That's the "fail fast" rule — better to die
in the first second than serve broken responses later.

---

## 1. `server.js` — the entry point

**What it does:** wires together the HTTP server, the WebSocket server, and
the background worker; handles graceful shutdown.

Step by step:
1. `const server = http.createServer(app)` — raw Node HTTP server wrapping
   our Express app. (Why not `app.listen()` directly? Because the WS library
   needs the raw server to handle the HTTP-Upgrade handshake on the same
   port.)
2. `signalWs.init(server)` — attach the WebSocket server on path `/ws`.
3. `SignalService.setBroadcast(signalWs.broadcastSignal)` — **dependency
   injection**: hand the broadcast function to SignalService so it can push
   over WS without importing the ws module directly. Tests can pass a stub.
4. `candleCloseHandler.start()` — subscribe to Redis. Wrapped in
   `try/catch` because in dev Redis may not be running; we'd rather start the
   API up (so devs can hit endpoints) than crash.
5. `server.listen(env.port, ...)` — begin serving requests.
6. Signal handlers: `SIGTERM` and `SIGINT` (Ctrl+C) call `shutdown(signal)`
   which calls `server.close(...)` — Node stops accepting new connections,
   lets in-flight requests finish, then `process.exit(0)`. A 10s safety
   timeout forces exit 1 if anything refuses to close.
7. `unhandledRejection` → log and keep running. `uncaughtException` → log and
   `exit(1)` (state is unknown, let the orchestrator — the tool that runs and
   auto-restarts the app, e.g. PM2/Docker/Kubernetes — restart it fresh).

### Gotchas
- The whole file is ~84 lines and does **nothing business-y** — that
  separation is on purpose: `server.js` = boot, `app.js` = HTTP, everything
  real is in `src/`.

---

## 2. `src/app.js` — the Express application

**What it does:** assembles the middleware pipeline and mounts the routes.

In order of execution (top-to-bottom = first-to-last):

1. **CORS** (`cors({...})`):
   - `origin: env.corsOrigins` — explicit allowlist (list of permitted
     websites) from `.env`. Never `origin: true` with `credentials: true`
     (that would trust ANY website and let a malicious one make
     cookie-authenticated calls as the victim — a "CSRF" attack). See
     topic 4.4.
   - `credentials: true` — cookies allowed.
   - Methods + allowed headers explicitly listed.
2. **`express.json({ limit: '1mb' })`** — parses JSON bodies, rejects > 1 MB.
3. **`express.urlencoded({ extended: true })`** — form bodies (rarely used
   here but harmless).
4. **`cookie-parser`** — populates `req.cookies` from the `Cookie` header so
   `auth.js` can read it.
5. **Request logger** — measures duration and logs every request except
   `/health` (health checks are loud and uninteresting).
6. **`GET /health`** — un-authed "liveness" endpoint (liveness = a simple
   "are you alive?" check that monitoring tools ping). Returns
   `{ status, service, uptime, timestamp }`. Used by load balancers / Docker /
   Kubernetes to know the app is up.
7. **`app.use('/api', routes)`** — all real endpoints under `/api`.
8. **404 handler** — anything that didn't match returns
   `{ success:false, error:{ code:'NOT_FOUND' } }`.
9. **`errorHandler`** — the special 4-argument error middleware (last so it
   catches anything thrown earlier).

### Gotchas
- The order matters. If you put the 404 before `/api`, every API request
  would 404. If you put `errorHandler` first, it would never fire (Express
  only calls it when `next(err)` was called by an earlier handler).

---

## 3. `src/config/env.js` — environment validation

**What it does:** loads `.env` once, refuses to start if anything required is
missing, and exports a typed `env` object so the rest of the code never
reads `process.env` directly.

1. `dotenv.config({ path: path.resolve(__dirname, '../../.env') })` — load
   the file relative to **this** file, not the cwd. Means the app works no
   matter where you run it from.
2. `required` array → loop → `console.error` + `process.exit(1)` if any are
   missing. **Fail fast.**
3. Builds the `env` object: `appDb`, `marketDb`, `redis`, `corsOrigins` (CSV
   split + trim + filter empty), `jwt`.
4. `corsOrigins` defaults to `http://localhost:3000` if `CORS_ORIGINS` isn't
   set — useful in dev, locked-down in prod.

### Gotchas
- `Number(process.env.PORT) || 3003` — string env vars always need explicit
  conversion.
- `marketDb` and `appDb` share host/port/user/password — only the **database
  name** differs. Both pools point at the same Postgres server.

---

## 4. `src/config/constants.js` — THE SINGLE SOURCE OF TRUTH

**What it does:** defines timeframes, indicators, operators, actions, modes.
Every other file imports from here.

Key tables:

```js
TIMEFRAME_MAP = {
  '1m':  { ohlcTable: '"OHLC_1MIN"',  indicatorTable: 'feature_table_1min'  },
  '5m':  { ohlcTable: '"OHLC_5MIN"',  indicatorTable: 'feature_table_5min'  },
  ...
};

INDICATOR_CONFIG = {
  'SMA (10)': { column: 'sma_10', category: 'Technical Indicator' },
  'SMA (20)': { column: 'sma_20', category: 'Technical Indicator' },
};
// VARIABLE_TO_COLUMN and INDICATOR_COLUMNS are AUTO-derived from INDICATOR_CONFIG below.
```

Derived constants (computed at load time):
- `VALID_TIMEFRAMES` = `Object.keys(TIMEFRAME_MAP)`.
- `TF_ORDER` = explicit order to define "smallest first" (for picking the
  driver timeframe).
- `VARIABLE_TO_COLUMN` — built from `INDICATOR_CONFIG` so adding an
  indicator updates everything at once.
- `INDICATOR_COLUMNS` — column names used by `indicator.model.upsert`.
- `VALID_OPERATORS`, `VALID_CATEGORIES`, `VALID_ACTIONS`, `VALID_MODES` —
  validator allowlists.

### Gotchas
- To add an indicator, the original author left this rule in the panel's
  CLAUDE.md: **one line in `INDICATOR_CONFIG` + one TA-Lib call in
  `IndicatorService.compute()`** + (eventually) a DB column on the
  co-developer's side.
- Comments mention scaffolding for RSI/EMA/MACD/Bollinger/ATR — they wait
  for DB columns. The frontend's `constants/strategy.js` mirrors only what
  the engine actually supports today.

---

## 5. `src/config/logger.js` — Winston

**What it does:** configures the logger used everywhere.

- Level: `debug` in dev, `info` in prod.
- Format pipeline: timestamps + error stack capture, then either:
  - dev: colorized human-readable single line:
    `2026-06-11 10:30:01 [info]: Indicators computed for RELIANCE on 5m {"ts": "..."}`
  - prod: JSON line per log (structured for log collectors).
- Transport: Console only.

### Gotchas
- The dev `printf` pulls `timestamp, level, message, stack` then
  **`...meta`** — extra fields you pass become JSON appended at the end.
  This is what makes `logger.error('X', { error: err.message })` clean.

---

## 6. `src/db/marketDb.js` and `src/db/appDb.js`

**What they do:** lazy singletons returning a `pg.Pool` for each database.

`getMarketPool()`:
1. First call: builds a new `Pool` with `env.marketDb` + `max: 20`,
   `idleTimeoutMillis: 30000`, `connectionTimeoutMillis: 10000`.
2. Adds `error` and `connect` listeners (logged via Winston).
3. Caches in `marketPool` (closure variable). Every subsequent call returns
   the same pool.

`getAppPool()` is identical but points at `victory_db`.

### Gotchas
- The pool is **never explicitly closed** during normal operation. On
  shutdown, `server.close()` lets requests drain; the pool's idle timeout
  closes connections naturally. (If we wanted bulletproof cleanup we'd call
  `pool.end()` in shutdown — currently not done.)
- `appDb` is exported but currently unused — every model uses
  `getMarketPool` (see topic 6.16 / architecture file).

---

## 7. `src/db/redis.js` — Redis subscriber + publisher

**What it does:** singletons for Redis connections.

- `getSubscriber()` — connects via `ioredis(env.redis.url)`. Used by
  `candleCloseHandler` to `subscribe('ohlc:candle_close', ...)`. A subscriber
  connection can ONLY subscribe — that's why it's separate.
- `getPublisher()` — for future use (when the engine publishes its own
  events).

### Gotchas
- `ioredis` auto-reconnects with "backoff" if Redis goes down (backoff =
  it waits a little longer before each retry instead of hammering nonstop).
  We don't need custom reconnect logic.
- The `try/catch` around `candleCloseHandler.start()` in `server.js` is
  there because the **subscriber's** connect happens lazily on the first
  `.subscribe()` call — if Redis isn't there, that throws.

---

## 8. `src/middleware/auth.js` — JWT verification

**What it does:** every authenticated route runs this. It extracts a JWT
from one of two places, verifies it, and attaches `req.user`.

```js
let token = null;
if (req.headers.authorization?.startsWith('Bearer ')) token = ...;
else if (req.cookies?.token)                          token = req.cookies.token;

if (!token) return next(new AuthError('Authentication token missing'));

const decoded = jwt.verify(token, env.jwt.secret);
req.user = { id: decoded.userId, userId: decoded.userId, permissions: decoded.permission || [] };
next();
```

### Gotchas
- **Header is preferred over cookie.** Allows API clients (curl, Postman)
  to override the cookie if needed.
- **No query-string token for HTTP** (see topic 4.5) — URLs end up in logs.
- The verify call **throws** on invalid/expired tokens; the catch wraps it
  in `AuthError` (401) so the central error handler can format it.
- Two aliases (`id` and `userId`) are exposed on `req.user` for backwards
  compatibility with any code that picks either.

---

## 9. `src/middleware/validate.js` — Joi runner

```js
const validate = (schema, source = 'body') => (req, res, next) => {
    const { error, value } = schema.validate(req[source], {
        abortEarly: false,   // collect ALL errors, not just the first
        stripUnknown: true,  // delete fields the schema doesn't know
    });
    if (error) {
        const details = error.details.map(d => ({ field: d.path.join('.'), message: d.message }));
        return next(new ValidationError('Validation failed', details));
    }
    req[source] = value;  // replace with cleaned/defaulted/coerced version
    next();
};
```

### Gotchas
- Replacing `req.body`/`req.query` after validation is what makes
  controllers safe: they only ever see clean data.
- `stripUnknown: true` is why undeclared filter fields disappear silently —
  the comment in `signal.validator.js` (the `signal_type` story) documents
  this exact trap.

---

## 10. `src/middleware/errorHandler.js` — central error formatter

```js
const errorHandler = (err, req, res, next) => {
    if (err.isOperational) logger.warn(err.message, { code, statusCode, path });
    else                    logger.error('Unhandled error', { message, stack, path });

    const statusCode = err.statusCode || 500;
    const message    = err.isOperational ? err.message : 'Internal server error';
    const code       = err.isOperational ? (err.code || 'INTERNAL_ERROR') : 'INTERNAL_ERROR';

    return res.status(statusCode).json({
        success: false,
        error: { message, code, ...(err.details ? { details: err.details } : {}) },
    });
};
```

### Gotchas
- **Stack traces never leave the server** — they go to logs only.
- `err.details` (used by ValidationError) carries the per-field messages
  so the frontend can highlight specific inputs.
- The 4-argument signature is what makes Express treat this as the error
  middleware (don't remove `next` even though we don't call it).

---

## 11. `src/routes/index.js` and the feature routers

`routes/index.js` mounts the three sub-routers and adds one stand-alone
route for "get the latest indicator row":

```js
router.use('/strategies', strategiesRoutes);   // /api/strategies/...
router.use('/signals',    signalsRoutes);      // /api/signals/...
router.use('/config',     configRoutes);       // /api/config/...
router.get('/indicators/:timeframe/:symbol', auth, signalsCtrl.getLatestIndicators);
```

### 11a. `routes/strategies.routes.js`

```js
GET    /         auth + validate(listStrategiesSchema, 'query') → ctrl.getAll
GET    /:id      auth                                            → ctrl.getById
POST   /         auth + validate(createStrategySchema)           → ctrl.create
PUT    /:id      auth + validate(updateStrategySchema)           → ctrl.update
DELETE /:id      auth                                            → ctrl.remove
PATCH  /:id/toggle  auth                                         → ctrl.toggle
POST   /:id/backtest auth                                        → ctrl.backtest
```

Note `getById` has no Joi schema — `:id` is just used as a SQL parameter and
the ownership check handles bad ids. `backtest` does its own validation in
the controller (date range etc.) because Joi-ifying that would be heavy and
the controller is the only caller.

### 11b. `routes/signals.routes.js`

```js
GET /             auth + validate(getSignalsSchema, 'query')     → ctrl.getAll
GET /strategy/:id auth + validate(strategySignalsSchema, 'query') → ctrl.getByStrategy
```

### 11c. `routes/config.routes.js`

```js
GET /strategy-options  auth → configCtrl.getStrategyOptions
```

---

## 12. Controllers

Controllers are tiny: parse `req`, call models/services, send response. They
never contain SQL or trading logic.

### 12a. `controllers/strategies.controller.js`

- **`getAll`** — pulls `page/limit/symbol/is_active` from query, builds the
  filters object, calls `strategyModel.findAll`. Note `is_active === 'true'`
  comparison (a string check — Joi keeps it as a string for that reason).
- **`getById`** — calls `findById`, throws `NotFoundError` on miss.
- **`create`** — copies `req.body`, attaches `user_id: req.user.id` (never
  trust a client-provided user id), then back-compat wraps single `symbol`
  into `[symbol]` array.
- **`update`** — the most complex one. Detects "rule logic changed"
  (action/conditions/exit_conditions/trailing_stop_loss/target). If yes,
  fetches currently TRIGGERED symbols BEFORE the update; then applies the
  update; then runs **Policy B**: for each triggered symbol, atomic
  CAS reset and synthetic EXIT signal. Each iteration is in its own
  try/catch — one bad signal must not block the others.
- **`remove`** — delete + 404 if nothing matched.
- **`toggle`** — flip `is_active`, then **force reset all symbols to IDLE**
  (so a re-activation starts cleanly).
- **`backtest`** — basic check for `start_date/end_date`, passes through to
  `BacktestService.run`.

### Gotchas
- The `update` keeps `oldStrategy` only when `triggeredSymbols.length > 0`
  to avoid an extra DB call when nothing needs Policy B.
- `oldStrategy.action || 'BUY'` — defensive default, because action could
  theoretically be null on very old rows.
- Look carefully at this line of `update`:
  `await SignalService.create({ ..., action_type: exitAction, signal_type: 'EXIT', ... })`.
  The action of the EXIT signal is the **flip** of the old strategy
  action — a BUY position is closed by SELL.

### 12b. `controllers/signals.controller.js`

- `getAll` — uppercases `symbol`, passes filters to `signalModel.findByUser`.
- `getByStrategy` — paginated list by strategy id.
- `getLatestIndicators` — used by `/api/indicators/:timeframe/:symbol`.
  Validates timeframe inline (the route has no Joi schema) and uppercases
  the symbol before querying.

### 12c. `controllers/config.controller.js`

Reads `INDICATOR_CONFIG` and returns:

```json
{
  "variableOptions": { "Technical Indicator": ["SMA (10)", "SMA (20)"] },
  "allIndicators":   ["SMA (10)", "SMA (20)"]
}
```

This is what the frontend's `StrategyForm` calls on mount (override of the
hard-coded fallback in `constants/strategy.js`).

---

## 13. Validators (Joi schemas)

### 13a. `validators/strategy.validator.js`

Defines:
- `conditionSchema` — every field, with `valid()` lists pulled from
  `constants.js`. `value` is conditional on `valueType` (when
  `IndicatorValue`, value must be a supported indicator name).
- `positionSizingSchema` — `fixed_quantity` (integer 1–100000) or
  `fixed_capital` (1–100,000,000).
- `brokerageConfigSchema` — `percent` type, `buy_pct`/`sell_pct` 0–5.
- `trailingStopLossSchema` — alternatives: fixed | percent | stepped, all
  with strict ranges (matched bit-for-bit in the engine's `parseTrailingStopLoss`).
- `targetSchema` — alternatives: percent (0.1–500) | price (>0).
- `createStrategySchema` — full create payload with `.or('symbols',
  'symbol')` (at least one must be present — back-compat with old
  single-symbol clients).
- `updateStrategySchema` — same fields but no `.required()` and `.min(1)`
  ensures at least one field is being updated.
- `listStrategiesSchema` — query string for the list endpoint with
  `page ≥ 1`, `limit ≤ 100`. The comment explains why `is_active` stays a
  string ('true'/'false') — the controller does `=== 'true'`.

### 13b. `validators/signal.validator.js`

`getSignalsSchema` — pagination + symbol + action_type + status +
**signal_type** (added because stripUnknown was eating it). All optional with
`allow('')` for "empty filter".

`strategySignalsSchema` — pagination only.

### Gotchas
- The `SUPPORTED_VARIABLES` list is derived from `VARIABLE_TO_COLUMN` so
  trying to save a strategy that uses an unsupported indicator (like RSI
  today) is rejected at save time with a clear error. Without this, the
  strategy would save but never fire.

---

## 14. Models — where ALL the SQL lives

### 14a. `models/strategy.model.js`

The biggest model. Key methods:

- **`findAll(userId, page, limit, filters)`** — paginated list with optional
  `symbol` and `is_active` filters. The `symbol` filter uses an `EXISTS`
  subquery so we don't break the GROUP BY shape. Two queries (count + page).
- **`findById(id, userId)`** — one strategy with its `symbols` array AND
  `symbol_states` (per-symbol trigger info). Uses `array_agg` AND `json_agg`
  together with the same LEFT JOIN.
- **`findActive(symbol, timeframe)`** — used for a single-timeframe lookup
  (legacy).
- **`findActiveBySymbol(symbol)`** — the hot path. INNER JOIN
  `strategies × strategy_symbols` for one symbol, returning all needed TSL
  state columns. This is what `ConditionEngine.evaluate` calls every candle.
- **`create(data)`** — transactional INSERT into `strategies` +
  `strategy_symbols` rows. JSONB configs (`position_sizing`,
  `brokerage_config`, `trailing_stop_loss`, `target`) become either NULL or
  `JSON.stringify(...)`. `ON CONFLICT DO NOTHING` on the junction means a
  duplicate symbol is harmless.
- **`update(id, userId, data)`** — dynamic SET clause: only fields present
  in `data` are updated. Then symbol diff: delete missing, insert new. All
  inside one transaction with COMMIT/ROLLBACK. Final SELECT returns the
  fully-joined view so the controller can send the updated strategy back.
- **`delete(id, userId)`** — simple DELETE + ownership check.
- **`toggleActive(id, userId)`** — `SET is_active = NOT is_active`, returns
  the new row via `findById`.
- **`countByUser(userId)`** — total/active/inactive counts with
  `COUNT(*) FILTER (WHERE …)`.
- **`transitionToTriggered(strategyId, symbol, entryExtremePrice)`** — the
  IDLE→TRIGGERED **atomic CAS** that fires ENTRY (topic 6.15 / 10.9). The
  WHERE includes `trigger_state = 'IDLE'` so a duplicate event becomes a
  no-op; `RETURNING strategy_id` + `rows.length > 0` is the success signal.
  Seeds `tsl_extreme_price = tsl_entry_price = driverClose` so the trailing
  logic has a baseline.
- **`resetTriggerState(strategyId, symbol)`** — the symmetric
  TRIGGERED→IDLE CAS (clears TSL columns).
- **`findTriggeredSymbols(strategyId)`** — used by Policy B to know which
  symbols need synthetic EXITs.
- **`forceResetTriggerState(strategyId)`** — non-conditional reset for ALL
  symbols (used by toggle).
- **`updateTslExtreme(strategyId, symbol, currentClose, isLong)`** —
  atomically lifts the extreme using `GREATEST`/`LEAST` inside the UPDATE.
  Returns extreme + entry + stepCount or null.
- **`updateTslStepCount(strategyId, symbol, newStepCount, newStopPrice)`** —
  guarded by `tsl_step_count < $3` so steps only move forward (race-safe).

### 14b. `models/signal.model.js`

- `create(data)` — INSERT into `signals`. Uses `??` (not `||`) for
  `trigger_price` so a real 0 is preserved.
- `findByUser(userId, page, limit, filters)` — paginated with optional
  `symbol/action_type/status/signal_type` filters, joining
  `strategy_name` from `strategies`. Counted with a separate `COUNT(*)`.
- `findByStrategy(strategyId, userId, page, limit)` — list one strategy's
  signals.
- `countToday(userId)` — `WHERE created_at >= CURRENT_DATE`.

### 14c. `models/indicator.model.js`

- `getOhlcCandles(symbol, timeframe, limit = 100)` — selects from the
  capital-cased `"OHLC_..."` table. Orders DESC + `LIMIT n`, then **reverses
  in JS** so the returned array is chronologically ascending (older →
  newer) — what TA-Lib expects.
- `upsert(timeframe, symbol, ts, values)` — builds a dynamic
  `INSERT ... ON CONFLICT (symbol, "timestamp") DO UPDATE`. Only columns
  present in `values` are written, so adding a new indicator later just
  adds another key to the object.
- `getLatest(symbol, timeframe)` / `getRecent(symbol, timeframe, limit=2)` —
  for live evaluation (`recent` returns 2 rows so cross-over operators can
  compare previous vs current).
- `getHistoricalRange(symbol, timeframe, start, end)` — for backtests.
  LEFT JOINs indicator rows with the OHLC table on `(symbol,
  "timestamp")` to attach a `close` per row.

### Gotchas
- The indicator and OHLC tables use double-quoted **capitalised** identifiers
  because that's how the data pipeline created them (topic 6.14).
- `getOhlcCandles` orders DESC + LIMIT + JS reverse. Doing
  `ORDER BY "Time" ASC LIMIT n` would give the WRONG window (oldest n
  candles instead of newest n).

---

## 15. Services — the brain

### 15a. `services/IndicatorService.js`

```js
async function compute(symbol, timeframe) {
    if (!talib) return null;                                  // graceful fallback
    if (candles.length < 20) return null;                     // SMA(20) needs ≥ 20
    const close = candles.map(r => Number(r.Close));
    const ts    = candles[candles.length - 1].Time;
    const values = {
        sma_10: last(talib.execute({ name:'SMA', inReal:close, optInTimePeriod:10 }).result.outReal),
        sma_20: last(talib.execute({ name:'SMA', inReal:close, optInTimePeriod:20 }).result.outReal),
    };
    await indicatorModel.upsert(timeframe, symbol, ts, values);
}
```

### Gotchas
- TA-Lib is a **native** library (C bindings). The `try/catch` around the
  `require` is there so a missing native build doesn't kill startup —
  indicators just don't compute. (Logs a warning so you know.)
- `optInTimePeriod` = lookback (the N in SMA(N)). TA-Lib returns one array
  shorter than the input — only the **last** value is what matters.
- `ts` = the close timestamp of the latest candle, used as the upsert key.

### 15b. `services/ConditionEngine.js` — the heart of the engine

This file is large. Here is what each piece does, in evaluation order:

**Helpers at the top:**
- `parseConditions(raw)` — handles both already-parsed and stringified JSONB.
- `resolveVarTimeframe(c, fb)` / `resolveValTimeframe(c, fb)` — picks the
  condition's own timeframe with a legacy fallback (`condition.timeframe`)
  and a final fallback to `strategy.timeframe || '1m'`.
- `collectTimeframes(conditions, fallback)` — union of all timeframes
  needed by the condition set (Time conditions excluded — they don't read
  indicators).
- `round2`, `pickDriverTimeframe`, `parseTrailingStopLoss`, `parseTarget`,
  `toISTMinutes`, `parseTimeValue`, `buildLiveIndicatorMap` — all small
  helpers used by `evaluate` and the per-condition evaluators.

**The main function: `evaluate(symbol)`**

```
strategies = strategy.model.findActiveBySymbol(symbol)
if empty: return                                  ← cheap exit

union of timeframes across all those strategies' conditions+exit_conditions
indicatorMap = buildLiveIndicatorMap(symbol, timeframes)
   For each tf: { current: latest indicator row, recent: last 2 rows }

for each strategy:
   try {
     if conditions.length === 0: continue       ← no entry rules, skip
     if strategy.trigger_state !== 'TRIGGERED':
        matched = evaluateConditions(entry conditions)
        if matched:
            pick driverTf, fetch latest 1-row candle there for entry price
            transitioned = transitionToTriggered (CAS)
            if !transitioned: continue          ← raced; another worker won
            SignalService.create({ signal_type:'ENTRY', action_type: strategy.action,
                                    trigger_price: driverClose,
                                    condition_snapshot: { conditions, indicator_values } })
     else:
        fetch currentClose ONCE if TSL or target is configured
        --- TSL block ---
        update extreme atomically
        compute stop level via tslMath
        if step advanced: updateTslStepCount (race-safe)
        if hit (long: close ≤ stop, short: close ≥ stop):
            resetTriggerState (CAS) → if win, EXIT signal
        --- TARGET block (only if TSL didn't exit) ---
        compute target level via targetMath (skip percent if no entry price)
        if hit: CAS reset + EXIT signal
        --- Exit conditions (only if neither TSL nor target exited) ---
        if exitConditions.length and evaluateConditions(exit):
            CAS reset + EXIT signal at primary-exit-tf's latest close
   } catch err {
       logger.error(`Error evaluating strategy ${strategy.id}`, ...)
       // one bad strategy must not block the others
   }
```

Visual of the per-strategy decision inside `evaluate`:

```
   for each active strategy on this symbol:
   ┌──────────────────────────────────────────────────────────────┐
   │            trigger_state == 'TRIGGERED' ?                     │
   └───────────────┬───────────────────────────┬──────────────────┘
              no   │                            │  yes
                   ▼                            ▼
       ┌───────────────────────┐    ┌──────────────────────────────────┐
       │ evaluate ENTRY conds  │    │ fetch driver close ONCE           │
       └──────────┬────────────┘    │   1. TSL hit?    → reset + EXIT   │
            match?│ yes             │   2. else TARGET hit? → reset+EXIT│
                  ▼                 │   3. else EXIT conds? → reset+EXIT│
       transitionToTriggered (CAS)  └──────────────────────────────────┘
                  │ won the race?
                  ▼ yes
       create ENTRY signal + WS push
```

**`evaluateConditions(conditions, map, fallback)`** — folds the boolean
results left-to-right using the **previous** condition's `logicalOperator`
(AND/OR). Returns `true` only if the final accumulator is exactly `true`
(undefined/false treated as false).

```
   conditions:  [ C1 ]  AND  [ C2 ]  OR  [ C3 ]
                   │           │          │
   result =      eval(C1)                 │
              AND eval(C2)   ◄── uses C1's logicalOperator
              OR  eval(C3)   ◄── uses C2's logicalOperator
              = ((C1 AND C2) OR C3)        evaluated strictly left → right
```

**`evaluateSingle(condition, map, fallback)`** — the per-condition logic:
- Time conditions route to `evaluateTimeCondition`.
- Indicator conditions: look up `varData` for the variable's timeframe,
  pull `column` from `VARIABLE_TO_COLUMN`, get `actualValue`. Then build
  `compareValue` (StaticValue = parse number; IndicatorValue = look up the
  other column on its own timeframe). Run the operator:
  - `>`, `<`, `>=`, `<=`, `=` (the `=` check allows a tiny gap of 0.0001 —
    called an "epsilon" — because decimal numbers on a computer are never
    perfectly exact, so `10.0000001` should still count as equal to `10`),
  - `Crosses Above` / `Crosses Below` — uses `varData.recent` to compare
    previous values too.

**`evaluateTimeCondition`** — IST minutes-since-midnight comparison
(`toISTMinutes` adds 330 to UTC minutes).

### Gotchas
- The engine fetches the candle close **once** per TRIGGERED iteration when
  either TSL or target is configured (the inline comment explains it halves
  the candle queries when both are configured).
- The driver timeframe is picked from `TF_ORDER.find(...)` — i.e. the
  smallest TF in use. This matters because TSL and target need fresh prices
  more than they need long context.
- A `Crosses Above` with no previous data (`recent.length < 1`) returns
  false (better than firing on bad data).
- Unknown variable name → `logger.warn` + false (the Joi validator should
  have prevented this, but defense in depth).

### 15c. `services/SignalService.js`

```js
let broadcastFn = null;
function setBroadcast(fn) { broadcastFn = fn; }
async function create(data) {
    const signal = await signalModel.create({ ...data, status: 'pending' });
    logger.info(`Signal created: ${data.action_type} ${data.symbol}`, { signalId, ... });
    if (broadcastFn) try { broadcastFn(data.user_id, { type:'NEW_SIGNAL', data: signal }); }
                      catch (err) { logger.error('WebSocket broadcast failed', ...); }
    return signal;
}
```

### Gotchas
- `setBroadcast` is the **dependency injection** point — `server.js` wires
  the WS broadcaster in at boot. This keeps `SignalService` from importing
  the ws layer, making it easier to test.
- A failing broadcast does NOT undo the DB row. The signal is recorded;
  only the live push failed. The frontend will see it on next page refresh.

### 15d. `services/BacktestService.js` — replaying history

The longest service. Structure:

1. **Helpers (top half)** — reused parsing for sizing/brokerage/TSL/target
   (mirroring the live `ConditionEngine` parsers exactly), and snapshot
   builders (`buildIndicatorSnapshot`, `traceConditionsForMatch`,
   `extractIndicatorColumns`) for the response.
2. **Math helpers** — `computeBrokerage`, `computeQuantity`,
   `computeRoundTripPL`, `computeExitFinancials`, `computePnLSummary`.
3. **`run(strategyId, userId, startDate, endDate, requestedSymbol)`** —
   - Ownership check via `findById`.
   - Pick the symbol (requested → strategy's symbols[0] → strategy.symbol),
     reject if not in the strategy's symbol list.
   - Parse + validate date range (≤ 30 days).
   - `collectTimeframes` over entry + exit conditions.
   - Fetch `historicalData[tf]` for each timeframe (`getHistoricalRange`).
   - Pick `driverTf` (smallest used).
   - Per-driver-candle loop: for each non-driver tf, advance its **pointer**
     while next row's ts ≤ current driver ts. Build `indicatorMap` for THIS
     candle. (This is the classic "walk two sorted lists together" trick. It's
     fast: it touches each row once — "O(N+M)" — instead of re-scanning every
     other-timeframe row for every candle, which would be the slow "O(N×M)".
     "Big-O" is just shorthand for how the work grows as the data grows.)

   ```
   Aligning a 1h indicator stream to a 5m driver loop (pointer never goes back):

   driver 5m:  09:05  09:10  09:15  ...  09:55  10:00  10:05
                 │      │      │            │      │      │
   1h rows:    09:00 ───────────────────► 09:00  10:00 ─────►
                 ▲                          ▲      ▲
                 └ pointer stays on 09:00   │      └ pointer advances to 10:00
                   until a newer 1h row     │        (its ts ≤ driver ts)
                   (10:00) is allowed       │
                                            each 5m candle sees the LATEST
                                            COMPLETED 1h row — never a future one.
   ```
   - Not in trade → run entry conditions → record ENTRY row (compute
     quantity/brokerage/total_amount when sizing is set; else skip
     position-sizing fields). Skip the trade if even one share doesn't fit
     the capital.
   - In trade → TSL → Target → exit conditions (same priority as live).
     Backtest uses the **stop/target level as the exit price** (theoretical),
     while live uses the candle close — small differences in numbers are
     expected.
4. Build `pnl_summary` and return `{ matches, summary, indicatorColumns,
   skipped_trades, pnl_summary }`.

### Gotchas
- The backtest re-implements its own TSL/target parsing instead of
  importing from `ConditionEngine`. The comment says "mirrors the live
  engine exactly" — keep them in sync if you change one.
- `pendingEntry` carries **two concerns**: exit-rule state (always
  populated) and execution/P&L state (only when sizing configured). The
  long comment in the code explains this carefully.
- `extractIndicatorColumns` builds the per-column metadata the frontend
  uses to render the indicator columns table. Duplicate columns are
  dropped via a Set.

---

## 16. Utils

### 16a. `utils/errors.js` — the error family (see topic 1.7).

### 16b. `utils/response.js` — `success`, `error`, `paginated` helpers.

### 16c. `utils/helpers.js` — `last`, `sleep`, `formatDate`, `parseJsonSafe`.

### 16d. `utils/tslMath.js` — `calculateTslStopLevel` (heavily commented).

### 16e. `utils/targetMath.js` — `calculateTargetLevel`.

Both math files were extracted so live + backtest **share one
implementation**. The comments at the top of each file are themselves part
of the documentation.

---

## 17. WebSocket — `src/ws/signalWs.js`

```js
const wss = new WebSocket.Server({ server, path: '/ws' });
const userConnections = new Map();  // userId → Set<ws>

wss.on('connection', (ws, req) => {
    const token = extractToken(req);       // query / cookie / header
    let userId = null;
    if (token) try { userId = jwt.verify(token, env.jwt.secret).userId; }
               catch { logger.warn('WS: invalid token'); }
    if (!userId) { ws.close(4401, 'Authentication required'); return; }

    userConnections.set(userId, userConnections.get(userId) || new Set());
    userConnections.get(userId).add(ws);

    ws.isAlive = true;
    ws.on('pong', () => ws.isAlive = true);
    ws.on('close', () => /* remove from set, delete entry if empty */);
    ws.on('error', (err) => logger.error(...));
    ws.send(JSON.stringify({ type:'CONNECTED', authenticated:true }));
});

setInterval(() => {                        // every 30s
    wss.clients.forEach(ws => {
        if (!ws.isAlive) return ws.terminate();
        ws.isAlive = false; ws.ping();     // ping; next sweep checks pong
    });
}, 30000);

function broadcastSignal(userId, payload) {
    const message = JSON.stringify(payload);
    const conns = userConnections.get(userId);
    if (conns) conns.forEach(ws => { if (ws.readyState === WebSocket.OPEN) ws.send(message); });
}
```

**The heartbeat, visualised:**

```
   every 30s sweep:
                        ┌─ ws.isAlive === false ? ──► ws.terminate()  (dead)
   for each client ws ──┤
                        └─ else: ws.isAlive = false; ws.ping()
                                                        │
                              browser auto-pongs ◄──────┘
                                  │
                          ws.on('pong') → ws.isAlive = true
                                  │
                          survives the NEXT sweep ✓
```

### Gotchas
- The heartbeat marks `isAlive = false` and pings; the **client's** TCP
  stack pongs automatically (no client code needed). If the next sweep sees
  it still false, the socket is dead → `terminate`.
- `wss.on('close')` clears the heartbeat — clean shutdown.
- `broadcastSignal` ONLY sends to the owning user. Per-user isolation is
  enforced by the registry — no shared "anonymous bucket" exists.

---

## 18. Background job — `src/jobs/candleCloseHandler.js`

```js
function start() {
    const subscriber = getSubscriber();
    subscriber.subscribe('ohlc:candle_close', errCb);
    subscriber.on('message', async (channel, message) => {
        if (channel !== 'ohlc:candle_close') return;
        let data; try { data = JSON.parse(message); } catch { /* log + return */ }
        const { symbol, timeframe, ts } = data;
        if (!symbol || !timeframe) { /* log + return */ }
        try {
            await IndicatorService.compute(symbol, timeframe);
            await ConditionEngine.evaluate(symbol);
        } catch (err) {
            logger.error(`candleCloseHandler failed for ${symbol} ${timeframe}`, ...);
        }
    });
}
```

### Gotchas
- Every error is caught and logged — a single bad message must NEVER kill
  the subscriber (it would silence the entire engine).
- `compute(symbol, timeframe)` is per-timeframe; `evaluate(symbol)` then
  looks at strategies on that symbol (which may use multiple timeframes
  including the one we just refreshed).
- Inside `evaluate`, the SAME-symbol same-timeframe events arrive in order
  (Redis pub/sub is per-connection ordered), so within one event we never
  race ourselves. Cross-event races are handled by the CAS pattern.

---

## 19. Migrations

`migrations/002_create_strategies.sql` — original `strategies` table.
`migrations/003_create_signals.sql` — original `signals` table.
`migrateTsl.js` — one-off Node script that added TSL columns to
`strategy_symbols`.

The README in `migrations/` says: these are **reference SQL**, not
auto-run. Apply to a fresh DB with `psql -f`. Tables actually used by the
engine today but NOT in these migrations include `strategy_symbols` itself
(added later — its SQL lives in the comments and in `migrateTsl.js`).

---

## 20. `.env` / `.env.example`

```
PORT=3003
NODE_ENV=development

PG_HOST=, PG_PORT=5432, PG_USER=, PG_PASSWORD=
PG_APP_DATABASE=victory_db
PG_MARKET_DATABASE=victory_market_db

REDIS_URL=redis://localhost:6379

JWT_SECRET=        ← MUST equal the main backend's secret
CORS_ORIGINS=      ← comma-separated allowlist (defaults to localhost:3000)
```

That's the entire backend. Every file you'll touch is one of the above.

**Next:** [04_FRONTEND_CONNECTION.md](04_FRONTEND_CONNECTION.md)
